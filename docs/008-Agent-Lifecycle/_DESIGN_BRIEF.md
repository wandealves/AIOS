---
Documento: _DESIGN_BRIEF (Fonte Única de Verdade Interna)
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0084, ADR-0085, ADR-0086, ADR-0087, ADR-0088, ADR-0089
RFCs relacionados: RFC-0001 (baseline), RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore — a propor)
Depende de: 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 010-Memory, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit, 027-Cluster
---

# Design Brief — Módulo 008 · Agent Lifecycle

> **Este documento é a FONTE ÚNICA DE VERDADE INTERNA do módulo 008.** Todos os 26
> documentos obrigatórios (`Vision.md` … `RFC.md`) DEVEM ser gerados a partir dele
> e NÃO PODEM contradizê-lo. Componentes, estados, campos, eventos, códigos de erro
> e chaves de configuração aqui definidos são canônicos. Contratos transversais
> (URN, envelope de evento CloudEvents, envelope de erro RFC 7807, `Idempotency-Key`,
> correlação `traceparent`/`X-AIOS-Tenant`) NÃO são redefinidos aqui — são
> **referenciados** de `../003-RFC/RFC-0001-Architecture-Baseline.md`.

As palavras-chave **DEVE**, **NÃO DEVE**, **DEVERIA**, **NÃO DEVERIA**, **PODE**
seguem RFC 2119 / RFC 8174.

---

## 0. Identidade e faixas reservadas do módulo

| Recurso | Faixa/valor reservado para o módulo 008 |
|---------|------------------------------------------|
| Domínio de subject NATS | `agent`, entidade `lifecycle` → `aios.<tenant>.agent.lifecycle.<acao>` |
| Domínio de código de erro | `AIOS-LIFECYCLE-<NNNN>` (faixa **0001–0099** reservada a este módulo) |
| Faixa de ADR | **ADR-0080 … ADR-0089** (008 × 10 = 0080; evita colisão com 009 → 0090+) |
| Tipo de recurso URN (RFC-0001 §5.1) | `agent` (compartilhado com 006/007); tipos internos `checkpoint`, `migration` propostos ao registro `004-API/Resources.md` |
| Prefixo de métricas | `aios_lifecycle_<nome>_<unidade>` |
| Pacote gRPC | `aios.lifecycle.v1` |
| Caminho REST | `/v1/agents/**` (subrecursos de ciclo de vida) |

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Responsabilidades (o módulo 008 É dono de)

O módulo 008 é o **Lifecycle Manager** do AIOS — análogo a `init`/`systemd` de um
SO clássico (ver `../000-Vision/Vision.md` §7.1). Ele DEVE:

- **R1.** Ser a autoridade única sobre a **máquina de estados canônica** do agente
  (`Created → Ready → Running ↔ Suspended → Hibernated → Migrating → Terminated/Failed`)
  e o **único** componente autorizado a persistir transições de estado do ACB.
- **R2.** Manter o **Agent Control Block (ACB)** como fonte da verdade do estado de
  ciclo de vida (identidade, estado, `generation`, ponteiros de memória/contexto,
  placement, cotas de referência, política de referência). O Kernel (006) é dono da
  *criação* do ACB e das *syscalls cognitivas*; o módulo 008 é dono do *estado de
  ciclo de vida* dentro do ACB e de seu versionamento por `generation`.
- **R3.** Executar **spawn / materialização** de agentes (frios ou novos) com
  `p99 ≤ 250 ms`, negociando slot de execução com o Scheduler (009) e o Runtime
  Supervisor (007).
- **R4.** Executar **suspensão** (`suspend`) e **retomada** (`resume`) de agentes
  quentes, preservando memória de trabalho.
- **R5.** Executar **hibernação** (`hibernate`) de *cold agents*: serializar
  ACB + memória de trabalho para MinIO/PostgreSQL, liberar RAM e registrar o agente
  como frio, permitindo `10⁶` agentes com poucos ativos em RAM.
- **R6.** Executar **checkpoint** e **restore** de ACB + memória de trabalho, como
  base de recuperação (RTO ≤ 15 min) e de migração.
- **R7.** Orquestrar **migração** de agentes entre shards/nós, consumindo decisões
  de placement do Scheduler (009) e do Cluster (027).
- **R8.** Emitir os eventos de domínio `aios.<tenant>.agent.lifecycle.*` de forma
  transacional (outbox) e idempotente.
- **R9.** Reconciliar continuamente **estado desejado × estado observado** do agente
  (padrão controller/reconcile), corrigindo derivas (ex.: runtime morto sem
  `Terminated`).
- **R10.** Gerenciar **leases/locks distribuídos** por agente para garantir
  transição serial (no máximo um coordenador por agente por vez).
- **R11.** Expor a **superfície de API** (REST + gRPC) de ciclo de vida e a consulta
  do histórico de transições (event-sourced).
- **R12.** Aplicar **retenção e expurgo** (tombstone) de agentes terminados,
  suportando o direito ao esquecimento (LGPD/GDPR) sobre snapshots/checkpoints.

### 1.2 Não-Responsabilidades (o módulo 008 NÃO É dono de)

O módulo 008 **NÃO DEVE**:

- **N1.** Executar o **loop cognitivo** do agente (ReAct, uso de tools) — isso é do
  **Agent Runtime (007)**. O 008 apenas inicia/pausa/encerra o processo de runtime.
- **N2.** Decidir **quem executa, quando e onde** (prioridade, custo, admissão,
  placement) — isso é do **Scheduler (009)**. O 008 *aplica* a decisão de placement,
  não a *toma*.
- **N3.** Expor **syscalls cognitivas** (`remember`, `recall`, `plan`, `invoke_tool`)
  nem gerenciar **cotas** — isso é do **Kernel (006)** e do **Cost Optimizer (026)**.
  O 008 lê cotas por referência para admitir uma materialização.
- **N4.** Persistir ou consolidar **camadas de memória** (working/short/long/semantic/
  procedural/episodic) — isso é do **Memory (010)**. O 008 serializa apenas o
  *ponteiro/snapshot da working memory* como parte do checkpoint, delegando a
  materialização da memória de longo prazo ao 010.
- **N5.** Tomar **decisões de política** (RBAC/ABAC) — isso é do **Policy Engine (022)**.
  O 008 atua como **PEP** (Policy Enforcement Point) chamando o PDP.
- **N6.** Gerenciar **provisão física de nós/pods**, autoscaling de infraestrutura ou
  replicação de storage — isso é do **Cluster/HA/DR (027)** e **Deployment (028)**.
- **N7.** Rotear **modelos LLM** (017) nem executar **tools** (015) diretamente.
- **N8.** Ser trilha de auditoria — o 008 *emite* eventos; a trilha imutável é do
  **Audit (025)**.

---

## 2. Decomposição em Componentes Internos

Todos os nomes em `PascalCase`. O módulo segue o padrão canônico da arquitetura
global (`../001-Architecture/Architecture.md` §5): *Gateway → PEP → estado →
coordenador → eventos*.

| Componente | Responsabilidade | Depende de (interno/externo) |
|------------|------------------|------------------------------|
| **LifecycleApiSurface** | Fachada REST/gRPC do módulo; validação de contrato, `Idempotency-Key`, versionamento, mapeamento de erro RFC 7807. | LifecyclePolicyEnforcer; LifecycleCoordinator |
| **LifecyclePolicyEnforcer** | PEP: intercepta toda mutação e consulta o PDP do Policy Engine (022); *default deny*. | Policy Engine (022) via gRPC |
| **LifecycleCoordinator** | Orquestrador central; recebe comandos, adquire lease, invoca a FSM, dispara efeitos (spawn/checkpoint/migrate) via saga, publica eventos. | StateMachineEngine; LeaseManager; AcbStore; todos os *Managers*; LifecycleEventPublisher |
| **StateMachineEngine** | Implementa a FSM canônica; valida transições, avalia guardas, calcula ações de entrada/saída; puramente determinístico e testável. | AcbStore (leitura) |
| **AcbStore** | Repositório do Agent Control Block e do histórico de transições (event-sourced + projeção de estado atual). CQRS: escrita append-only, leitura por projeção. | PostgreSQL; Redis (hot ACB) |
| **SpawnManager** | Materializa agente novo ou frio: negocia slot (009), pede runtime (007), aplica snapshot, aquece contexto/memória (010/011). Alvo `p99 ≤ 250 ms`. | Scheduler (009); Runtime Supervisor (007); SnapshotStore; WarmPoolManager |
| **WarmPoolManager** | Mantém pool de runtimes pré-aquecidos por tenant/shard para reduzir cold-start; políticas de reserva mínima e escala. | Runtime Supervisor (007); Redis |
| **HibernationController** | Detecta agentes ociosos (idle), aplica política de hibernação, dispara checkpoint, libera RAM, marca agente como *cold*. | CheckpointService; SnapshotStore; AcbStore |
| **CheckpointService** | Cria/restaura checkpoints consistentes de ACB + working memory; garante atomicidade, hash de integridade e cifragem. | SnapshotStore; Memory (010) |
| **SnapshotStore** | Persistência do estado serializado: blob em MinIO (payload) + metadados/índice em PostgreSQL; níveis de storage (hot/cold). | MinIO; PostgreSQL |
| **MigrationOrchestrator** | Saga de migração entre shards/nós (checkpoint → transfer → materializa no destino → invalida origem), com corte de escrita e drenagem. | CheckpointService; SpawnManager; Cluster (027); Scheduler (009) |
| **LeaseManager** | Locks/leases distribuídos por `agent_id`; garante um único coordenador por agente; renovação e fencing token. | Redis (SET NX PX + fencing) |
| **ReconciliationController** | Loop de reconciliação estado-desejado × observado; detecta runtime morto, checkpoint órfão, migração travada; aciona compensação. | AcbStore; Runtime Supervisor (007); LifecycleCoordinator |
| **LifecycleEventPublisher** | Publica eventos `agent.lifecycle.*` via **Outbox transacional** + JetStream; garante at-least-once e dedupe por `event.id`. | PostgreSQL (outbox); NATS/JetStream (020) |
| **LifecycleEventConsumer** | Consome eventos de 009 (placement), 007 (heartbeat/exit), 006 (ACB criado), 014 (task idle) para acionar transições reativas. | NATS/JetStream (020) |
| **TombstoneManager** | Aplica retenção de agentes `Terminated/Failed`, expurgo de snapshots/checkpoints (LGPD), *anti-resurrection* (IDs não reutilizados). | SnapshotStore; AcbStore; Audit (025) |
| **LifecycleTelemetry** | Instrumentação OTel (traces/metrics/logs) transversal aos componentes; expõe métricas `aios_lifecycle_*`. | OpenTelemetry (024) |

### 2.1 Diagrama de Componentes (ASCII)

```
                         REST / gRPC (aios.lifecycle.v1)
                                     │
                       ┌─────────────▼──────────────┐
                       │     LifecycleApiSurface     │  validação · idem-key · RFC7807
                       └─────────────┬──────────────┘
                                     │
                       ┌─────────────▼──────────────┐   consulta PDP
                       │   LifecyclePolicyEnforcer   │─────────────────▶ Policy Engine (022)
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
    │  PostgreSQL (ACB, transitions, outbox, snapshot idx) · Redis(hot)│
    │  MinIO (snapshot/checkpoint blobs)                               │
    └───┬──────────────────────────────────────────────────────┬──────┘
        │ Outbox                                                │
   ┌────▼──────────────────┐                        ┌───────────▼─────────────┐
   │ LifecycleEventPublisher│──▶ NATS/JetStream ────│  LifecycleEventConsumer  │
   │  aios.<tenant>.agent.  │    (020)              │  ← 009 placement         │
   │  lifecycle.*           │                       │  ← 007 heartbeat/exit    │
   └───────────────────────┘                        │  ← 006 acb.created       │
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

## 3. Modelo de Dados / Entidades Canônico

Todas as entidades multi-tenant DEVEM carregar `tenant_id` e usar URN conforme
RFC-0001 §5.1 (`urn:aios:<tenant>:<tipo>:<id>`, `<id>` = ULID). IDs de agente são
compartilhados com 006/007; o módulo 008 acrescenta `generation` (contador monotônico
de encarnação incrementado a cada materialização/restore/migração).

### 3.1 `AgentControlBlock` (ACB) — projeção de estado atual

| Campo | Tipo | Chave/Índice | Descrição |
|-------|------|--------------|-----------|
| `agent_id` | ULID (URN `agent`) | PK | Identidade estável do agente. |
| `tenant_id` | slug (`a-z0-9-`) | PK composta · RLS | Fronteira de isolamento (RFC-0001 §5.1). |
| `state` | enum `LifecycleState` | idx | Estado canônico atual (§4). |
| `generation` | int64 | — | Encarnação atual; incrementa a cada spawn/restore/migração. |
| `desired_state` | enum `LifecycleState` | idx | Estado-alvo (reconciliação). |
| `priority_class` | enum {`system`,`interactive`,`batch`,`best_effort`} | idx | Referência de prioridade (definida por 009). |
| `placement_shard` | int | idx | `hash(tenant_id,agent_id) mod N` (RFC/Arch §12). |
| `placement_node` | string | idx | Nó/host atual do runtime (quando quente). |
| `runtime_instance_id` | ULID \| null | — | Instância de Agent Runtime (007) associada. |
| `working_memory_ref` | URN \| null | — | Ponteiro para working memory (010). |
| `context_ref` | URN \| null | — | Ponteiro para janela de contexto (011). |
| `policy_ref` | URN | — | Política aplicável (022). |
| `quota_ref` | URN | — | Referência de cotas (006/026). |
| `last_checkpoint_id` | ULID \| null | fk → Checkpoint | Último checkpoint válido. |
| `cold_since` | timestamptz \| null | idx parcial | Início da hibernação (se `Hibernated`). |
| `wake_count` | int32 | — | Número de materializações a partir de frio. |
| `fencing_token` | int64 | — | Token de lease para escrita segura. |
| `created_at` | timestamptz | — | Criação do ACB (origem: 006). |
| `updated_at` | timestamptz | — | Última mutação de estado. |
| `terminated_at` | timestamptz \| null | — | Instante do estado terminal. |
| `failure_code` | string \| null | — | `AIOS-LIFECYCLE-<NNNN>` da última falha. |
| `labels` | jsonb | GIN | Metadados livres (agent group, etc.). |

Invariantes: (i) `(tenant_id, agent_id)` único; (ii) `state` terminal ⇒ `desired_state`
terminal e `runtime_instance_id = null`; (iii) `state = Hibernated ⇒ last_checkpoint_id ≠ null`.

### 3.2 `LifecycleTransition` (event-sourced, append-only)

| Campo | Tipo | Chave/Índice | Descrição |
|-------|------|--------------|-----------|
| `transition_id` | ULID | PK | Identidade da transição (também `event.id`). |
| `agent_id` | ULID | idx | Agente. |
| `tenant_id` | slug | idx · RLS | Isolamento. |
| `seq` | int64 | idx único `(agent_id,seq)` | Ordem monotônica por agente. |
| `from_state` | enum | — | Estado de origem. |
| `to_state` | enum | — | Estado de destino. |
| `trigger` | enum `LifecycleTrigger` | — | `spawn`,`ready`,`start`,`suspend`,`resume`,`hibernate`,`wake`,`migrate`,`terminate`,`fail`,`reconcile`. |
| `actor` | string | — | Origem (`api:<sub>`, `scheduler`, `reconciler`, `runtime`). |
| `guard_result` | jsonb | — | Guardas avaliadas e resultado. |
| `reason` | text | — | Motivo/detalhe. |
| `generation` | int64 | — | Encarnação no momento. |
| `correlation` | jsonb | — | `traceparent`, `X-AIOS-Tenant`, `idempotency_key`. |
| `created_at` | timestamptz | idx | Instante. |

### 3.3 `Checkpoint`

| Campo | Tipo | Chave/Índice | Descrição |
|-------|------|--------------|-----------|
| `checkpoint_id` | ULID (URN `checkpoint`) | PK | Identidade. |
| `agent_id` | ULID | idx | Agente. |
| `tenant_id` | slug | idx · RLS | Isolamento. |
| `generation` | int64 | — | Encarnação capturada. |
| `storage_uri` | string | — | `minio://<bucket>/<tenant>/<agent>/<ckpt>.zst`. |
| `size_bytes` | int64 | — | Tamanho do blob comprimido. |
| `content_hash` | string (sha-256) | idx | Integridade. |
| `codec` | enum {`json+zstd`,`msgpack+zstd`} | — | Formato de serialização. |
| `encryption` | enum {`aes-256-gcm`} | — | Cifragem em repouso (chave por tenant, 021). |
| `working_memory_ref` | URN | — | Ponteiro de working memory associado (010). |
| `consistency` | enum {`consistent`,`crash`} | — | `consistent` = quiesced; `crash` = sob falha. |
| `created_at` | timestamptz | idx | Criação. |
| `expires_at` | timestamptz \| null | idx | Retenção. |

### 3.4 `HibernationRecord`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `agent_id` / `tenant_id` | ULID / slug (PK · RLS) | Agente frio. |
| `cold_since` | timestamptz | Início da hibernação. |
| `storage_tier` | enum {`hot`,`warm`,`cold`} | Camada do snapshot. |
| `checkpoint_id` | ULID (fk) | Snapshot de materialização. |
| `last_wake_latency_ms` | int32 | Última latência de materialização. |
| `wake_count` | int32 | Materializações acumuladas. |

### 3.5 `MigrationJob`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `migration_id` | ULID (URN `migration`) (PK) | Identidade da saga. |
| `agent_id` / `tenant_id` | ULID / slug (idx · RLS) | Agente migrado. |
| `source_shard` / `source_node` | int / string | Origem. |
| `target_shard` / `target_node` | int / string | Destino (decisão 009/027). |
| `phase` | enum {`quiesce`,`checkpoint`,`transfer`,`materialize`,`cutover`,`cleanup`} | Fase da saga. |
| `status` | enum {`pending`,`running`,`succeeded`,`failed`,`compensated`} | Estado da migração. |
| `checkpoint_id` | ULID (fk) | Checkpoint de transferência. |
| `started_at` / `finished_at` | timestamptz | Janela. |

### 3.6 `LifecycleOutbox` / `Lease`

- `LifecycleOutbox`: `event_id` (ULID, PK), `tenant_id`, `subject`, `payload` (jsonb
  CloudEvents), `published` (bool), `created_at` — garante atomicidade DB+evento
  (padrão Outbox, `../001-Architecture/Architecture.md` §13).
- `Lease` (Redis): chave `aios:{tenant}:lc:lease:{agent_id}` → `{holder, fencing_token,
  expires_at}`; TTL curto com renovação; `fencing_token` monotônico persiste no ACB.

---

## 4. Máquina(s) de Estado Canônica(s)

### 4.1 Estados (`LifecycleState`)

| Estado | Descrição | RAM alocada? | Terminal? |
|--------|-----------|--------------|-----------|
| `Created` | ACB criado pelo Kernel (006); ainda não admitido. | não | não |
| `Ready` | Admitido pelo Scheduler (009); aguardando slot/execução. | reservada | não |
| `Running` | Runtime (007) executando o loop cognitivo. | sim | não |
| `Suspended` | Pausado; memória de trabalho preservada em RAM/Redis. | sim (reduzida) | não |
| `Hibernated` | *Cold agent*: estado serializado em MinIO/PG; **RAM liberada**. | não | não |
| `Migrating` | Em migração entre shards/nós (saga ativa). | transitória | não |
| `Terminated` | Encerrado normalmente; ACB tombstoned. | não | **sim** |
| `Failed` | Encerrado por falha irrecuperável. | não | **sim** |

### 4.2 Diagrama de Estados (ASCII)

```
                    spawn(new)
        ┌────────┐  [G1]      ┌────────┐  start[G3]   ┌─────────┐
   ●───▶│Created │───────────▶│ Ready  │─────────────▶│ Running │
        └────────┘   admit    └────────┘              └────┬────┘
             │       [G2]          ▲                       │  ▲
             │                     │ resume/wake[G5]       │  │ resume[G5]
             │ fail[G9]            │                  suspend│  │
             ▼                     │                    [G4] │  │
        ┌────────┐           ┌──────────┐  hibernate[G6]  ┌──▼──┴────┐
        │ Failed │◀──────────│Hibernated│◀────────────────│Suspended │
        └────────┘  fail[G9] └────┬─────┘   (idle)        └────┬─────┘
             ▲                    │ wake[G5]                    │
             │                    │  (materialize <250ms)       │
             │            migrate[G7]                    migrate[G7]
             │              ┌───────────────┐                   │
             └──────fail────│   Migrating   │◀──────────────────┘
                     [G9]   └───────┬───────┘
                                    │ cutover[G8]
                                    ▼   (→ Ready|Running|Hibernated no destino)
                             ┌────────────┐
              terminate[G10] │ Terminated │  ● (estado terminal)
      (de qualquer não-term.)└────────────┘
```

### 4.3 Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |
|---|-----------|---------|------------------------------|
| T1 | `∅ → Created` | `spawn`/`acb.created` (006) | **G0**: ACB válido, tenant autenticado, política permite `create`. |
| T2 | `Created → Ready` | `admit` (009) | **G1**: Scheduler admitiu (cotas/orçamento ok); política `run` permitida. |
| T3 | `Ready → Running` | `start` | **G3**: slot alocado, Runtime (007) provisionado, lease adquirido, `generation` incrementado. |
| T4 | `Running → Suspended` | `suspend` | **G4**: lease válido; ausência de operação crítica não-preemptível em curso. |
| T5 | `Suspended → Running` | `resume` | **G5**: slot disponível, working memory íntegra, política `run` ainda válida. |
| T6 | `Suspended → Hibernated` | `hibernate` | **G6**: idle ≥ `idle_ttl` **ou** pressão de RAM; checkpoint `consistent` gravado com sucesso. |
| T7 | `Hibernated → Ready/Running` | `wake`/`materialize` | **G5**: checkpoint íntegro (hash ok), Scheduler concede slot; `generation` incrementado. |
| T8 | `Running/Suspended → Migrating` | `migrate` | **G7**: decisão de placement de 009/027; lease adquirido; destino saudável. |
| T9 | `Migrating → Ready/Running/Hibernated` | `cutover` | **G8**: materialização no destino confirmada; origem drenada/invalidada. |
| T10 | `qualquer não-terminal → Terminated` | `terminate` | **G10**: política `terminate` permitida; efeitos colaterais drenados; checkpoint final opcional. |
| T11 | `qualquer não-terminal → Failed` | `fail` | **G9**: erro irrecuperável (timeout de saga, corrupção de checkpoint, runtime perdido além de `max_recovery`). |

Invariantes globais: **INV1** — uma transição por agente por vez (LeaseManager);
**INV2** — toda transição escreve `LifecycleTransition` **e** `LifecycleOutbox` na
mesma transação; **INV3** — estados terminais são absorventes (nenhuma transição de
saída); **INV4** — `Hibernated`/`Migrating` DEVEM possuir checkpoint `consistent`
válido antes de liberar RAM/origem.

---

## 5. Superfície de API (REST e gRPC)

Autenticação, versionamento, `Idempotency-Key`, correlação e envelope de erro
seguem RFC-0001 §5.4–§5.7. Toda mutação DEVE passar pelo `LifecyclePolicyEnforcer`.

### 5.1 REST (`/v1/agents`)

| Método | Caminho | Ação | Idempotência | Sucesso |
|--------|---------|------|--------------|---------|
| POST | `/v1/agents` | Cria/spawn agente (delegado ao 006 + admissão 009). | `Idempotency-Key` obrigatório | 202 Accepted |
| GET | `/v1/agents/{id}` | Lê ACB (estado atual). | — | 200 |
| GET | `/v1/agents/{id}/lifecycle` | Histórico de transições (paginado). | — | 200 |
| POST | `/v1/agents/{id}/suspend` | `Running → Suspended`. | `Idempotency-Key` | 202 |
| POST | `/v1/agents/{id}/resume` | `Suspended → Running`. | `Idempotency-Key` | 202 |
| POST | `/v1/agents/{id}/hibernate` | `Suspended → Hibernated`. | `Idempotency-Key` | 202 |
| POST | `/v1/agents/{id}/wake` | `Hibernated → Ready/Running` (materializa). | `Idempotency-Key` | 202 |
| POST | `/v1/agents/{id}/migrate` | Inicia migração (target opcional). | `Idempotency-Key` | 202 |
| POST | `/v1/agents/{id}/checkpoint` | Cria checkpoint sob demanda. | `Idempotency-Key` | 201 |
| POST | `/v1/agents/{id}/restore` | Restaura de checkpoint (`checkpointId`). | `Idempotency-Key` | 202 |
| DELETE | `/v1/agents/{id}` | `→ Terminated`. | `Idempotency-Key` | 202 |
| GET | `/v1/agents/{id}/checkpoints` | Lista checkpoints. | — | 200 |

### 5.2 gRPC (`aios.lifecycle.v1.LifecycleService`)

```
service LifecycleService {
  rpc Spawn(SpawnRequest)          returns (LifecycleAck);
  rpc Suspend(AgentRef)            returns (LifecycleAck);
  rpc Resume(AgentRef)             returns (LifecycleAck);
  rpc Hibernate(AgentRef)          returns (LifecycleAck);
  rpc Wake(WakeRequest)            returns (LifecycleAck);   // materialize
  rpc Migrate(MigrateRequest)      returns (LifecycleAck);
  rpc Checkpoint(AgentRef)         returns (CheckpointRef);
  rpc Restore(RestoreRequest)      returns (LifecycleAck);
  rpc Terminate(TerminateRequest)  returns (LifecycleAck);
  rpc GetState(AgentRef)           returns (AgentControlBlock);
  rpc StreamTransitions(AgentRef)  returns (stream LifecycleTransition);
}
```

Chamadas internas de outros módulos usam gRPC; chamadas externas passam pelo YARP
(REST). gRPC mapeia erros para `google.rpc.Status` + `ErrorInfo` com o mesmo `code`
(RFC-0001 §5.4).

### 5.3 Catálogo de códigos de erro (`AIOS-LIFECYCLE-<NNNN>`, faixa 0001–0099)

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-LIFECYCLE-0001` | 404 | não | Agente não encontrado (`agent_id`/`tenant_id`). |
| `AIOS-LIFECYCLE-0002` | 409 | não | Transição inválida para o estado atual (viola FSM §4). |
| `AIOS-LIFECYCLE-0003` | 409 | sim | Lease não adquirido (outra transição em curso). |
| `AIOS-LIFECYCLE-0004` | 423 | não | Agente em estado terminal (`Terminated`/`Failed`). |
| `AIOS-LIFECYCLE-0005` | 429 | sim | Admissão negada pelo Scheduler (cota/capacidade). |
| `AIOS-LIFECYCLE-0006` | 504 | sim | Timeout de materialização (spawn/wake > alvo). |
| `AIOS-LIFECYCLE-0007` | 422 | não | Checkpoint corrompido (hash divergente). |
| `AIOS-LIFECYCLE-0008` | 404 | não | Checkpoint inexistente para restore. |
| `AIOS-LIFECYCLE-0009` | 409 | sim | Migração já em andamento para o agente. |
| `AIOS-LIFECYCLE-0010` | 502 | sim | Destino de migração indisponível/insalubre. |
| `AIOS-LIFECYCLE-0011` | 500 | sim | Falha de serialização/deserialização do snapshot. |
| `AIOS-LIFECYCLE-0012` | 403 | não | Política negou a operação (PDP `deny`). |
| `AIOS-LIFECYCLE-0013` | 409 | não | `generation`/`fencing_token` obsoleto (escrita rejeitada). |
| `AIOS-LIFECYCLE-0014` | 503 | sim | Storage de snapshot (MinIO/PG) indisponível. |
| `AIOS-LIFECYCLE-0015` | 500 | não | Runtime perdido e irrecuperável (→ `Failed`). |

---

## 6. Catálogo de Eventos NATS

Subjects seguem `aios.<tenant>.<dominio>.<entidade>.<acao>` (RFC-0001 §5.3);
envelope CloudEvents (RFC-0001 §5.2); produção via Outbox at-least-once; consumidores
idempotentes por `event.id`.

### 6.1 Eventos emitidos (produtor: módulo 008)

| Subject (`aios.<tenant>.agent.lifecycle.<acao>`) | `type` | Quando |
|---|---|---|
| `…lifecycle.created` | `aios.agent.lifecycle.created` | ACB registrado no módulo (após 006). |
| `…lifecycle.ready` | `aios.agent.lifecycle.ready` | Admitido (Scheduler). |
| `…lifecycle.running` | `aios.agent.lifecycle.running` | Runtime iniciou o loop. |
| `…lifecycle.suspended` | `aios.agent.lifecycle.suspended` | Agente suspenso. |
| `…lifecycle.resumed` | `aios.agent.lifecycle.resumed` | Agente retomado. |
| `…lifecycle.hibernated` | `aios.agent.lifecycle.hibernated` | Cold agent; RAM liberada. |
| `…lifecycle.woken` | `aios.agent.lifecycle.woken` | Materializado de frio. |
| `…lifecycle.migrating` | `aios.agent.lifecycle.migrating` | Saga de migração iniciada. |
| `…lifecycle.migrated` | `aios.agent.lifecycle.migrated` | Cutover concluído no destino. |
| `…lifecycle.checkpointed` | `aios.agent.lifecycle.checkpointed` | Checkpoint criado. |
| `…lifecycle.restored` | `aios.agent.lifecycle.restored` | Restore concluído. |
| `…lifecycle.terminated` | `aios.agent.lifecycle.terminated` | Encerramento normal. |
| `…lifecycle.failed` | `aios.agent.lifecycle.failed` | Falha irrecuperável. |

`data` mínimo de cada evento: `{ agentId(URN), tenantId, generation, fromState,
toState, trigger, reason?, checkpointId?, placement{shard,node}? }` — versionado por
`dataschema` (`aios://schemas/agent.lifecycle.<acao>/1`).

### 6.2 Eventos consumidos (consumidor: módulo 008)

| Subject consumido | Produtor | Ação disparada |
|---|---|---|
| `aios.<tenant>.agent.control.created` | Kernel (006) | Inicializa ACB → `Created`. |
| `aios.<tenant>.scheduler.placement.decided` | Scheduler (009) | `Created→Ready` (admissão) ou alvo de migração. |
| `aios.<tenant>.scheduler.preemption.requested` | Scheduler (009) | `Running→Suspended` (preempção). |
| `aios.<tenant>.agent.runtime.heartbeat` | Runtime Supervisor (007) | Reconciliação de liveness. |
| `aios.<tenant>.agent.runtime.exited` | Runtime Supervisor (007) | `→Terminated`/`Failed` conforme código. |
| `aios.<tenant>.task.execution.completed` | Workflow/Task (014) | Sinal de idle → candidato a hibernação. |
| `aios.<tenant>.cluster.node.draining` | Cluster (027) | Dispara migração em massa do nó. |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito (DEVE) | Critério de aceite |
|----|------------------|--------------------|
| FR-001 | Manter e transicionar a FSM canônica (§4). | Toda transição válida registrada; inválida rejeitada com `AIOS-LIFECYCLE-0002`. |
| FR-002 | Spawn de agente novo com negociação de slot (009) e runtime (007). | Agente atinge `Running` com evento `running` emitido. |
| FR-003 | Suspender/retomar preservando working memory. | `resume` recupera estado idêntico ao pré-`suspend` (hash de working memory igual). |
| FR-004 | Hibernar cold agents liberando RAM. | Após `hibernate`, RSS do runtime = 0; snapshot íntegro em MinIO/PG. |
| FR-005 | Materializar cold agent sob demanda. | `wake` restaura `generation+1` e retoma execução; evento `woken`. |
| FR-006 | Checkpoint/restore consistentes de ACB + working memory. | `restore` reconstrói ACB e working memory a partir de checkpoint `consistent`. |
| FR-007 | Migrar agente entre shards/nós via saga. | `MigrationJob` conclui `cutover`; origem invalidada; evento `migrated`. |
| FR-008 | Emitir todos os eventos `agent.lifecycle.*` transacionalmente. | Nenhum evento perdido (outbox) ; dedupe por `event.id`. |
| FR-009 | Reconciliar estado desejado × observado. | Runtime morto sem `Terminated` é detectado e reconciliado ≤ `reconcile_interval`. |
| FR-010 | Serializar transições por agente (lease). | Duas transições concorrentes: uma sucede, outra recebe `AIOS-LIFECYCLE-0003`. |
| FR-011 | Aplicar retenção/expurgo (tombstone/LGPD) de agentes terminados. | Purge apaga snapshots e emite evento auditável (025); ID não reutilizado. |
| FR-012 | Expor histórico de ciclo de vida (event-sourced) por API. | `GET …/lifecycle` retorna sequência ordenada por `seq`. |
| FR-013 | Ser PEP para toda mutação (default deny). | Requisição sem autorização do PDP resulta em `AIOS-LIFECYCLE-0012`. |
| FR-014 | Manter WarmPool para reduzir cold-start. | Reserva mínima por tenant/shard mantida conforme config. |

### 7.2 Requisitos Não-Funcionais (NFR) com metas numéricas

| ID | Atributo | SLO/Meta | SLI (como medir) |
|----|----------|----------|------------------|
| NFR-001 | Performance — spawn/materialização | `p99 ≤ 250 ms`, `p50 ≤ 80 ms` (cold→Running, com WarmPool) | histograma `aios_lifecycle_spawn_duration_ms` |
| NFR-002 | Performance — decisão de transição | `p99 ≤ 20 ms` (excl. efeitos I/O) | `aios_lifecycle_transition_duration_ms` |
| NFR-003 | Performance — checkpoint | `p99 ≤ 500 ms` para working memory ≤ 64 MiB | `aios_lifecycle_checkpoint_duration_ms` |
| NFR-004 | Escalabilidade | Suportar `≥ 10⁶` agentes por cluster com `≤ 5%` ativos em RAM simultaneamente | `aios_lifecycle_agents_total{state}` |
| NFR-005 | Recuperabilidade | **RTO ≤ 15 min** para restaurar agente após falha de nó | tempo `runtime.exited` → `woken`/`resumed` |
| NFR-006 | Durabilidade / RPO | **RPO ≤ 5 min** de working memory (checkpoint periódico + em suspend) | intervalo entre checkpoints válidos |
| NFR-007 | Disponibilidade do serviço 008 | `≥ 99,95%` (stateless, replicado) | uptime de readiness |
| NFR-008 | Throughput de transições | `≥ 50.000 transições/s` por cluster | `rate(aios_lifecycle_transitions_total)` |
| NFR-009 | Integridade de snapshot | `0` restore de checkpoint corrompido não detectado | verificação de `content_hash` |
| NFR-010 | Latência de migração | `p99 ≤ 2 s` (agente ≤ 64 MiB, mesmo DC) sem perda de trabalho aceito | `aios_lifecycle_migration_duration_ms` |
| NFR-011 | Overhead de hibernação em RAM | ACB frio ocupa `≤ 4 KiB` em índice quente (Redis) | tamanho de projeção fria |
| NFR-012 | Observabilidade | `100%` das transições com trace OTel + evento + log correlacionado | cobertura de `traceparent` |
| NFR-013 | Idempotência | Repetição de mutação com mesmo `Idempotency-Key` ⇒ mesmo resultado, `0` efeito duplicado | teste de contrato |

---

## 8. Chaves de Configuração Principais

Convenção de variável de ambiente: `AIOS_LIFECYCLE_<CHAVE>`. Escopo: `global`,
`por-tenant` (override) ou `por-agente` (label). Recarga em runtime quando indicada.

| Chave | Default | Faixa válida | Escopo | Recarregável |
|-------|---------|--------------|--------|--------------|
| `spawn.target_p99_ms` | `250` | `50–2000` | global/tenant | sim |
| `spawn.timeout_ms` | `1500` | `250–10000` | global/tenant | sim |
| `warmpool.min_per_shard` | `4` | `0–256` | tenant/shard | sim |
| `warmpool.max_per_shard` | `64` | `0–1024` | tenant/shard | sim |
| `hibernation.idle_ttl_s` | `300` | `30–86400` | tenant/agente | sim |
| `hibernation.ram_pressure_threshold_pct` | `85` | `50–99` | global | sim |
| `hibernation.storage_tier` | `warm` | `hot\|warm\|cold` | tenant | sim |
| `checkpoint.interval_s` | `120` | `10–3600` | tenant/agente | sim |
| `checkpoint.codec` | `msgpack+zstd` | `json+zstd\|msgpack+zstd` | global | não |
| `checkpoint.max_working_memory_mib` | `256` | `1–4096` | tenant | sim |
| `checkpoint.encryption` | `aes-256-gcm` | fixo | global | não |
| `lease.ttl_ms` | `5000` | `1000–30000` | global | sim |
| `lease.renew_ms` | `1500` | `500–10000` | global | sim |
| `reconcile.interval_ms` | `2000` | `500–60000` | global | sim |
| `reconcile.runtime_dead_after_ms` | `15000` | `3000–120000` | global | sim |
| `migration.max_concurrent_per_node` | `8` | `1–128` | global/node | sim |
| `migration.saga_timeout_ms` | `120000` | `10000–600000` | global | sim |
| `recovery.rto_budget_s` | `900` | `60–3600` | global | sim |
| `retention.terminated_ttl_days` | `30` | `1–3650` | tenant | sim |
| `retention.checkpoint_ttl_days` | `7` | `1–365` | tenant | sim |
| `outbox.publish_batch` | `256` | `1–2048` | global | sim |

---

## 9. Modos de Falha e Estratégia de Recuperação

Todas as mutações são idempotentes (chave por `Idempotency-Key`/`event.id`) ou
compensáveis (saga). Retries com backoff exponencial + jitter; falhas persistentes
vão para DLQ (JetStream). Metas: **RTO ≤ 15 min**, **RPO ≤ 5 min** (NFR-005/006).

| Modo de falha (FMEA) | Detecção | Recuperação |
|----------------------|----------|-------------|
| Runtime (007) morre em `Running` | Ausência de heartbeat > `runtime_dead_after_ms` | ReconciliationController materializa do último checkpoint (RPO ≤ 5 min) → `Ready/Running`; se além de `max_recovery` → `Failed`. |
| Crash do coordenador (008) durante transição | Lease expira; transição incompleta | Novo coordenador relê `LifecycleTransition` + saga log; retoma ou compensa; `fencing_token` invalida escritas do coordenador antigo. |
| Checkpoint corrompido | `content_hash` divergente no restore | Rejeita (`AIOS-LIFECYCLE-0007`); tenta checkpoint anterior; se nenhum → `Failed` com auditoria. |
| MinIO/PG indisponível | Erro de I/O no SnapshotStore | Retry com backoff; hibernação adiada (mantém quente); `AIOS-LIFECYCLE-0014`; alerta. |
| Migração travada (nó destino cai) | `saga_timeout_ms` excedido | Compensa: invalida destino, reativa origem (agente nunca perde trabalho aceito); `MigrationJob=compensated`. |
| Split-brain (dois coordenadores) | `fencing_token` inconsistente | Escrita com token obsoleto rejeitada (`AIOS-LIFECYCLE-0013`); apenas token mais novo persiste. |
| Evento duplicado | `event.id` já processado | Consumidor deduplica (idempotência RFC-0001 §5.5); no-op. |
| Storm de hibernação (RAM crítica) | `ram_pressure_threshold_pct` | HibernationController prioriza por prioridade/idle; backpressure na admissão (009). |
| Outbox não publica (NATS down) | Lag de publicação | Retry do publisher; eventos persistem no outbox; nada é perdido (at-least-once). |

Degradação graciosa: sob storage indisponível, o módulo mantém agentes quentes e
suspende **novas** hibernações/migrações, preservando a FSM em memória + PG.

---

## 10. Escalabilidade e Concorrência (rumo a milhões de agentes)

- **Sharding determinístico**: `shard = hash(tenant_id, agent_id) mod N`
  (`../001-Architecture/Architecture.md` §12). Cada shard tem coordenadores próprios;
  o estado (ACB/transitions) é particionado por shard no PostgreSQL.
- **Agentes majoritariamente frios**: em `10⁶` agentes, `≤ 5%` ativos em RAM
  (NFR-004). Cold agents vivem em MinIO/PG; índice quente mínimo (`≤ 4 KiB`, NFR-011)
  em Redis para roteamento e wake rápido (`< 250 ms`, NFR-001).
- **WarmPool** por tenant/shard elimina cold-start de processo de runtime; a
  materialização é apenas *deserialização + attach de memória*, não boot completo.
- **Concorrência serial por agente** via LeaseManager (Redis `SET NX PX` + fencing
  token), sem lock global — escala horizontalmente por shard.
- **Backpressure**: admissão de materializações limitada por
  `migration.max_concurrent_per_node` e por sinais do Scheduler (009); JetStream
  aplica pressão nos consumidores. Storm de wake é enfileirado, não derruba o nó.
- **CQRS**: escrita append-only de transições; leitura por projeção `AgentControlBlock`
  (cache quente em Redis) — escala leitura de estado sem tocar o log.
- **Outbox + JetStream** desacopla publicação de eventos do caminho quente de
  transição, permitindo throughput `≥ 50.000 transições/s` (NFR-008).
- **Migração em lote** dirigida por `cluster.node.draining` respeita orçamento de
  concorrência por nó para não saturar rede/storage.
- **Locks**: nenhum lock distribuído global; apenas leases de granularidade por
  agente. Transições da FSM são operações curtas e determinísticas (NFR-002).

---

## 11. ADRs e RFCs a propor

### 11.1 ADRs específicas do módulo (faixa reservada ADR-0080 … ADR-0089)

| ADR | Título | Decisão-chave |
|-----|--------|---------------|
| ADR-0080 | Máquina de estados canônica do ciclo de vida do agente | Estados/transições/guardas de §4 como contrato imutável. |
| ADR-0081 | Cold Agent Hibernation: estado serializado em MinIO + índice em PostgreSQL | Onde e como hibernar para escalar a 10⁶. |
| ADR-0082 | Formato e codec de checkpoint (`msgpack+zstd`, AES-256-GCM) | Serialização, compressão e cifragem de ACB + working memory. |
| ADR-0083 | Materialização sob demanda com WarmPool (spawn < 250 ms) | Estratégia de cold-start e reserva de runtimes. |
| ADR-0084 | Concorrência por agente via lease + fencing token (Redis) | Serialização de transições sem lock global. |
| ADR-0085 | Migração como saga com compensação (checkpoint → transfer → cutover) | Migração entre shards/nós sem perda de trabalho aceito. |
| ADR-0086 | Event sourcing de transições + Outbox transacional | Proveniência e publicação atômica de eventos. |
| ADR-0087 | Reconciliação estado-desejado × observado (controller pattern) | Detecção/correção de deriva e recuperação de runtime. |
| ADR-0088 | Fronteira 008×006×009×007 (quem decide vs. quem aplica) | Delimita responsabilidades e contratos entre módulos. |
| ADR-0089 | Retenção, tombstone e expurgo LGPD de agentes/checkpoints | Direito ao esquecimento e não-reutilização de IDs. |

### 11.2 RFCs relacionadas

- **RFC-0001** (baseline, *Accepted*) — reutilizada integralmente (URN, eventos,
  erros, idempotência, correlação). NÃO redefinida.
- **RFC-0080** (a propor) — *Cold Agent Hibernation & Checkpoint/Restore Protocol*:
  formaliza o protocolo de serialização, materialização, versionamento por
  `generation` e semântica de consistência (`consistent`/`crash`), consolidando
  ADR-0081/0082/0083/0085.

---

## 12. Decisões de Segurança

### 12.1 AuthN / AuthZ

- **AuthN**: tokens OAuth2/OIDC validados no API Gateway (YARP, 021); claims
  assinadas propagadas; `X-AIOS-Tenant` DEVE bater com o tenant autenticado
  (RFC-0001 §6). Comunicação interna entre serviços via **mTLS** (021).
- **AuthZ**: `LifecyclePolicyEnforcer` é **PEP**; toda mutação consulta o **PDP**
  do Policy Engine (022) — *default deny*. Ações mapeadas a permissões:
  `lifecycle:spawn`, `:suspend`, `:resume`, `:hibernate`, `:wake`, `:migrate`,
  `:checkpoint`, `:restore`, `:terminate`, `:read`.
- Nenhum serviço aceita `tenant` divergente do contexto (isolamento por URN/subject).

### 12.2 Threat Model STRIDE (resumido)

| Ameaça | Vetor no módulo | Mitigação |
|--------|-----------------|-----------|
| **S**poofing | Chamador finge outro tenant/agente | OIDC + mTLS; `X-AIOS-Tenant` validado; URN escopado por tenant. |
| **T**ampering | Alteração de checkpoint/ACB em repouso | `content_hash` (sha-256) + AES-256-GCM; RLS no PG; buckets MinIO por tenant. |
| **R**epudiation | Negar ter suspendido/terminado um agente | Event sourcing + trilha imutável (025); `actor` em cada transição. |
| **I**nformation disclosure | Vazamento de working memory em snapshot | Cifragem por tenant (021); minimização; `detail` de erro não vaza PII (RFC-0001 §6). |
| **D**enial of Service | Storm de wake/spawn esgota RAM | Backpressure + WarmPool limitado + admissão do Scheduler; leases evitam thundering herd. |
| **E**levation of privilege | Transição sem autorização | PEP/PDP default-deny; `AIOS-LIFECYCLE-0012`; fencing token contra coordenador rogue. |

### 12.3 LGPD / GDPR

- Snapshots/checkpoints com working memory PODEM conter dados pessoais → cifrados
  em repouso, com metadados de base legal e retenção (RFC-0001 §7; 010/025).
- **Direito ao esquecimento**: `TombstoneManager` executa expurgo rastreável de ACB,
  transições, checkpoints e snapshots ao término de retenção ou sob solicitação,
  emitindo evento auditável (025). IDs **NÃO DEVEM** ser reutilizados (RFC-0001 §5.1).
- Minimização: payloads de evento `agent.lifecycle.*` carregam apenas metadados de
  estado — **NÃO DEVEM** incluir conteúdo de memória de trabalho.

---

## 13. Rastreabilidade para os 26 documentos

Este brief alimenta diretamente: `Architecture.md` (§2), `ClassDiagrams.md` (§2/§3),
`Database.md` (§3), `StateMachine.md` (§4), `API.md` (§5), `Events.md` (§6),
`FunctionalRequirements.md`/`NonFunctionalRequirements.md` (§7), `Configuration.md`
(§8), `FailureRecovery.md` (§9), `Scalability.md` (§10), `ADR.md`/`RFC.md` (§11),
`Security.md` (§12). Nenhum documento gerado PODE contradizer as decisões acima.

*Fim do Design Brief — Módulo 008 · Agent Lifecycle.*
