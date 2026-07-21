---
Documento: Design Brief Interno (Fonte Única de Verdade do Módulo)
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0008, ADR-0010 (globais); ADR-0060..ADR-0069 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline); RFC-0006 (Cognitive Syscall ABI, a propor); RFC-0007 (ACB & Quota Model, a propor)
Depende de: 001-Architecture, 022-Policy (PDP), 009-Scheduler, 008-Lifecycle/Agent-Registry, 020-Communication (NATS), 021-Security, 024-Observability, 025-Audit
---

# 006-Kernel — Design Brief (Fonte Única de Verdade)

> **Escopo deste documento.** Este brief é a fonte única de verdade **interna** do
> módulo 006-Kernel. Os 26 documentos obrigatórios (ver `README.md` e
> `../_templates/MODULE_TEMPLATE.md`) DEVEM ser derivados deste brief e **NÃO PODEM
> contradizê-lo**. Onde um contrato central já existe (URN, envelope de evento,
> envelope de erro, idempotência, correlação, subjects), este documento **referencia**
> a `../003-RFC/RFC-0001-Architecture-Baseline.md` e **não o redefine**.

> **Palavras normativas** conforme RFC 2119 / RFC 8174: DEVE, NÃO DEVE, DEVERIA,
> NÃO DEVERIA, PODE.

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Missão

O Kernel é o **núcleo cognitivo** do AIOS. Ele é o análogo do kernel de um SO
clássico (ver Glossário: *Kernel Cognitivo*, *ACB*, *Syscall Cognitiva*): expõe uma
**ABI de syscalls cognitivas**, mantém o **Agent Control Block (ACB)** como estrutura
autoritativa de cada agente, aplica **capability enforcement** como PEP consultando o
PDP do `022-Policy`, contabiliza e faz *enforcement* de **cotas de recurso**, e
**coordena o ciclo de vida** do agente delegando escalonamento ao `009-Scheduler` e
materialização ao `008-Lifecycle`.

O Kernel é o **único ponto de entrada** privilegiado pelo qual um agente (via
`007-Agent-Runtime`) solicita qualquer recurso cognitivo do AIOS. É a fronteira de
confiança entre o loop de raciocínio (plano de dados) e os serviços governados
(plano de controle).

### 1.2 Responsabilidades (o Kernel DEVE)

| # | Responsabilidade |
|---|------------------|
| R-01 | Expor a **superfície de syscalls cognitivas** (verbos): `spawn`, `kill`, `suspend`, `resume`, `remember`, `recall`, `plan`, `invoke_tool`, `route_model`, `get_quota`, `checkpoint` — via REST (externo) e gRPC (interno). |
| R-02 | Manter o **ACB Store** como registro autoritativo do estado de controle de cada agente (identidade URN, estado, prioridade, cotas, ponteiros de memória/contexto, política, timestamps). |
| R-03 | Atuar como **PEP (Policy Enforcement Point)**: para toda syscall privilegiada, montar o contexto de decisão e consultar o **PDP** do `022-Policy` antes de executar. *Default deny*. |
| R-04 | Contabilizar e fazer *enforcement* de **cotas** (tokens, cpu-ms, memória, custo USD, ferramentas, syscalls/s) por agente e por tenant, com reserva/consumo/liberação atômicos. |
| R-05 | Coordenar o **ciclo de vida** do agente (máquina de estados canônica do ACB), delegando admissão/placement ao `009-Scheduler` e materialização/hibernação ao `008-Lifecycle`. |
| R-06 | **Emitir eventos** de ciclo de vida e de decisão no barramento NATS no padrão `aios.<tenant>.agent.lifecycle.*` e `aios.<tenant>.agent.syscall.*`, com envelope CloudEvents da RFC-0001. |
| R-07 | Garantir **idempotência** de toda mutação (via `Idempotency-Key`) e **atomicidade** entre persistência de estado e publicação de evento (padrão Outbox). |
| R-08 | Roteirizar syscalls de recurso para os módulos donos do recurso (Memory `010`, Context `011`, Planning `012`, Tool `015`, Model Router `017`) sem executar a lógica de negócio deles — o Kernel é **broker governado**, não executor. |
| R-09 | Produzir **telemetria OTel** (traces/metrics/logs) e **registro de auditoria** (via `025-Audit`) para toda ação privilegiada e toda decisão de política. |
| R-10 | Aplicar **isolamento multi-tenant**: rejeitar qualquer `tenant` divergente do contexto autenticado; propagar `tenant` em URNs, subjects e claims. |

### 1.3 Não-Responsabilidades (o Kernel NÃO DEVE)

| # | Não-Responsabilidade | Dono real |
|---|----------------------|-----------|
| NR-01 | Executar o loop de raciocínio (ReAct) do agente ou hospedar o sandbox. | `007-Agent-Runtime` |
| NR-02 | Decidir **onde/quando** executar (placement, preempção, admission de fila). O Kernel *pede* um slot; o Scheduler decide. | `009-Scheduler` |
| NR-03 | Materializar/hibernar fisicamente o processo do agente (start/stop de runtime, snapshot de estado em MinIO). | `008-Lifecycle` + Runtime Supervisor |
| NR-04 | **Decidir** política (ser PDP). O Kernel é PEP; a decisão é do `022-Policy`. | `022-Policy` |
| NR-05 | Armazenar/recuperar itens de memória, comprimir contexto, planejar, invocar ferramentas ou chamar LLM diretamente. Ele apenas **encaminha** essas syscalls, aplicando cota + capability. | `010`, `011`, `012`, `015`, `017` |
| NR-06 | Persistir a trilha de auditoria imutável (o Kernel emite; o Audit persiste). | `025-Audit` |
| NR-07 | Gerenciar identidade federada, emissão/validação de tokens OAuth2/OIDC (feito no Gateway/`021`). O Kernel consome claims já validadas. | `021-Security` + Gateway |
| NR-08 | Definir orçamento de custo por tenant (o Kernel *aplica* a cota; o orçamento é definido por `026-Cost-Optimizer`). | `026-Cost-Optimizer` |

---

## 2. Decomposição em Componentes Internos

### 2.1 Componentes (PascalCase · responsabilidade · dependências)

| Componente | Responsabilidade | Depende de (interno → externo) |
|------------|------------------|-------------------------------|
| **SyscallGateway** | Recebe syscalls (REST/gRPC), valida schema/versão, extrai correlação (`traceparent`, `X-AIOS-Tenant`, `Idempotency-Key`), aplica rate-limit por agente, despacha ao handler correto. É o *front door* da ABI. | CapabilityEnforcer, IdempotencyStore, resource brokers; ← Gateway/YARP |
| **CapabilityEnforcer** | PEP. Para cada syscall privilegiada monta o `DecisionRequest` (sujeito, ação, recurso, ambiente) e consulta o PDP via PolicyClient; aplica *default deny*; cacheia decisões com TTL curto. | PolicyClient, AcbStore |
| **PolicyClient** | Cliente resiliente do `022-Policy` (PDP): gRPC com circuit breaker, cache de decisões, *fail-closed* configurável, carga de *policy bundles* versionados. | → 022-Policy |
| **AcbStore** | CRUD autoritativo do ACB; projeção quente em Redis + fonte da verdade em PostgreSQL; controle de concorrência otimista (`version`); leitura *read-your-writes*. | → PostgreSQL, Redis |
| **ResourceQuotaManager** | Reserva/consumo/liberação atômicos de cotas (tokens, cpu-ms, mem, custo, tools, syscalls/s) por agente e tenant; janelas *token-bucket*/*leaky-bucket*; sinaliza `AIOS-QUOTA-*`. | → Redis (contadores), PostgreSQL (limites), 026 (limites) |
| **LifecycleCoordinator** | Implementa a máquina de estados do ACB; orquestra `spawn/suspend/resume/kill/checkpoint` como sagas; pede slot ao Scheduler e materialização ao Lifecycle; garante idempotência e compensação. | AcbStore, SchedulerClient, LifecycleClient, EventEmitter |
| **SchedulerClient** | Cliente do `009-Scheduler`: submete pedido de admissão/placement, recebe slot/decisão de preempção; timeout + retry idempotente. | → 009-Scheduler |
| **LifecycleClient** | Cliente do `008-Lifecycle`/Runtime Supervisor: solicita materialização, hibernação, snapshot/restore de estado. | → 008-Lifecycle |
| **ResourceBrokerRouter** | Encaminha syscalls de recurso (`remember`, `recall`, `plan`, `invoke_tool`, `route_model`) ao módulo dono, após capability+cota; aplica bulkhead e circuit breaker por dependência. | ResourceQuotaManager, CapabilityEnforcer; → 010/011/012/015/017 |
| **IdempotencyStore** | Persiste resultado por `Idempotency-Key` (≥24h, RFC-0001 §5.5); deduplica repetições; garante *exactly-once* efetivo em mutações. | → Redis (quente) + PostgreSQL (durável) |
| **EventEmitter** | Publica eventos CloudEvents no NATS/JetStream via **Outbox transacional**; garante at-least-once + ordenação por stream; deduplicação por `event.id`. | → PostgreSQL (outbox), NATS/JetStream |
| **CheckpointManager** | Coordena `checkpoint`: coleta ponteiros de memória/contexto do ACB, dispara snapshot durável (via Lifecycle→MinIO) e registra o *checkpoint id* no ACB para *resume* consistente. | AcbStore, LifecycleClient, EventEmitter |
| **KernelTelemetry** | Instrumentação transversal OTel (spans por syscall, métricas `aios_kernel_*`, logs Serilog correlacionados) e emissão de auditoria para `025`. | → 024-Observability, 025-Audit |

### 2.2 Diagrama de Componentes (ASCII)

```
                         REST (externo) / gRPC (interno)  · via API Gateway (YARP)
                                          │
   ┌──────────────────────────────────────▼──────────────────────────────────────┐
   │                          KERNEL SERVICE (006 · .NET 10)                       │
   │                                                                              │
   │   ┌───────────────────┐        ┌───────────────────┐                          │
   │   │  SyscallGateway   │───────▶│ IdempotencyStore  │◀── Redis + PostgreSQL    │
   │   │ (valida/despacha) │        └───────────────────┘                          │
   │   └─────────┬─────────┘                                                       │
   │             │ (toda syscall privilegiada)                                     │
   │             ▼                                                                  │
   │   ┌───────────────────┐   consulta PDP   ┌───────────────┐                     │
   │   │ CapabilityEnforcer│─────────────────▶│ PolicyClient  │──▶ 022-Policy (PDP) │
   │   │      (PEP)        │◀─── allow/deny ──│ (CB + cache)  │                     │
   │   └─────────┬─────────┘                  └───────────────┘                     │
   │             │ allow                                                            │
   │      ┌──────┴───────────────────────────┬───────────────────────────┐         │
   │      ▼                                   ▼                           ▼         │
   │ ┌─────────────────┐   ┌───────────────────────────┐   ┌───────────────────┐   │
   │ │ ResourceQuota   │   │  LifecycleCoordinator     │   │ ResourceBroker    │   │
   │ │ Manager         │◀─▶│ (FSM ACB · sagas)         │   │ Router            │   │
   │ │ (reserva/consumo)│   └───┬──────────────┬────────┘   │ remember/recall/  │   │
   │ └───────┬─────────┘       │              │            │ plan/invoke_tool/ │   │
   │         │        ┌────────▼───┐   ┌───────▼────────┐   │ route_model       │   │
   │         │        │SchedulerCli│   │ LifecycleCli   │   └────────┬──────────┘   │
   │         │        └─────┬──────┘   └───────┬────────┘            │              │
   │  ┌──────▼─────────┐    │                  │        ┌────────────▼───────────┐  │
   │  │  AcbStore      │    │                  │        │ CheckpointManager      │  │
   │  │ (PG + Redis,   │    │                  │        └────────────┬───────────┘  │
   │  │  optimistic v) │    │                  │                     │              │
   │  └──────┬─────────┘    │                  │                     │              │
   │         │              │                  │        ┌────────────▼───────────┐  │
   │         └──────────────┴──────────────────┴───────▶│    EventEmitter        │  │
   │                                                     │  (Outbox → JetStream)  │  │
   │  ┌───────────────────────────────────────────────┐ └────────────┬───────────┘  │
   │  │ KernelTelemetry (OTel spans/metrics/logs + audit)│             │             │
   │  └───────────────────────────────────────────────┘              │             │
   └──────────────┬───────────────┬───────────────┬──────────────────┼─────────────┘
                  ▼               ▼               ▼                   ▼
            009-Scheduler   008-Lifecycle   010/011/012/015/017   NATS aios.<tenant>.agent.*
```

---

## 3. Modelo de Dados / Entidades Canônicas

> Alinhado à RFC-0001 §5.1 (URN `urn:aios:<tenant>:<tipo>:<id>`, `<id>` = ULID). Toda
> tabela DEVE ter `tenant_id` e Row-Level Security. Tipos são do PostgreSQL 16.

### 3.1 Entidade `AgentControlBlock` (ACB) — tabela `kernel.acb`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:agent:<ULID>` (RFC-0001). |
| `tenant_id` | `text` | NOT NULL, RLS | Fronteira de isolamento. |
| `agent_id` | `char(26)` | NOT NULL, UNIQUE(tenant_id,agent_id) | ULID puro (parte `<id>` do URN). |
| `state` | `text` | NOT NULL, CHECK ∈ AcbState | Estado da FSM (§4). |
| `priority` | `smallint` | NOT NULL, 0..9 (default 5) | Prioridade de escalonamento (0=maior). |
| `parent_urn` | `text` | NULL, FK→acb.urn | Agente que executou `spawn` (árvore de agentes). |
| `quota_ref` | `uuid` | NOT NULL, FK→kernel.quota.id | Cotas efetivas do agente. |
| `memory_ptr` | `jsonb` | NOT NULL | Ponteiros p/ namespaces de memória (`010`): `{working,short,long,semantic,episodic,procedural}`. |
| `context_ptr` | `jsonb` | NOT NULL | Ponteiro de janela de contexto/sessão (`011`). |
| `policy_ref` | `text` | NOT NULL | URN/versão do *policy binding* aplicável (`022`). |
| `capability_set` | `text[]` | NOT NULL default '{}' | Capabilities concedidas (cache do último *grant*). |
| `runtime_ref` | `text` | NULL | Handle do Agent Runtime materializado (`007`/`008`), NULL se *cold*. |
| `checkpoint_ref` | `text` | NULL | Último checkpoint durável (MinIO) para *resume*. |
| `shard_key` | `int` | NOT NULL, GENERATED | `hash(tenant_id, agent_id) mod N` (sharding). |
| `version` | `bigint` | NOT NULL default 0 | Concorrência otimista (OCC). |
| `created_at` | `timestamptz` | NOT NULL | Criação (RFC 3339 UTC). |
| `updated_at` | `timestamptz` | NOT NULL | Última mutação. |
| `last_active_at` | `timestamptz` | NULL | Última execução ativa (base p/ hibernação). |
| `terminated_at` | `timestamptz` | NULL | Preenchido no estado terminal. |
| `labels` | `jsonb` | NOT NULL default '{}' | Metadados livres (agent group, SLA class). |

Índices: `PK(urn)`; `UNIQUE(tenant_id, agent_id)`; `btree(tenant_id, state)`;
`btree(shard_key)`; `btree(tenant_id, last_active_at)` (varredura de hibernação).

### 3.2 Entidade `Quota` — tabela `kernel.quota`

| Campo | Tipo | Constraint | Descrição |
|-------|------|-----------|-----------|
| `id` | `uuid` | PK | Identidade do conjunto de cotas. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `scope` | `text` | CHECK ∈ {tenant, agent} | Nível de aplicação. |
| `subject_urn` | `text` | NOT NULL | URN do tenant ou do agente alvo. |
| `tokens_limit` | `bigint` | NOT NULL | Teto de tokens LLM na janela. |
| `cpu_ms_limit` | `bigint` | NOT NULL | Teto de cpu-ms na janela. |
| `mem_mib_limit` | `int` | NOT NULL | Teto de memória de trabalho. |
| `cost_usd_limit` | `numeric(12,4)` | NOT NULL | Orçamento (definido por `026`). |
| `tools_limit` | `int` | NOT NULL | Invocações de ferramentas por janela. |
| `syscalls_per_sec` | `int` | NOT NULL | Rate-limit de syscalls. |
| `window` | `interval` | NOT NULL default '1 hour' | Janela de reset. |
| `enforcement` | `text` | CHECK ∈ {hard, soft} | `hard`=rejeita; `soft`=marca e alerta. |
| `version` | `bigint` | NOT NULL | OCC. |

> **Contadores de consumo** (voláteis, alta frequência) vivem em **Redis** como
> *token-buckets* por `subject_urn`; PostgreSQL guarda apenas limites e o saldo
> reconciliado periodicamente (fonte da verdade dos limites, não dos contadores quentes).

### 3.3 Entidade `SyscallRecord` (event-sourced) — tabela `kernel.syscall_log`

| Campo | Tipo | Constraint | Descrição |
|-------|------|-----------|-----------|
| `id` | `char(26)` | PK (ULID) | Id da invocação de syscall. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `agent_urn` | `text` | NOT NULL | Agente chamador. |
| `verb` | `text` | NOT NULL | Um dos 11 verbos (§5). |
| `idempotency_key` | `text` | NULL, UNIQUE(tenant_id,idempotency_key) | RFC-0001 §5.5. |
| `decision` | `text` | CHECK ∈ {allow, deny} | Resultado do PEP. |
| `deny_code` | `text` | NULL | `AIOS-CAP-*`/`AIOS-QUOTA-*` quando `deny`. |
| `status` | `text` | CHECK ∈ {ok, error} | Resultado da execução. |
| `latency_ms` | `int` | NOT NULL | Latência observada. |
| `trace_id` | `text` | NOT NULL | Correlação OTel. |
| `created_at` | `timestamptz` | NOT NULL | Timestamp. |

### 3.4 Entidade `OutboxMessage` — tabela `kernel.outbox`

| Campo | Tipo | Constraint | Descrição |
|-------|------|-----------|-----------|
| `id` | `char(26)` | PK (ULID = `event.id`) | Id do evento CloudEvents. |
| `tenant_id` | `text` | NOT NULL | Isolamento. |
| `subject` | `text` | NOT NULL | Subject NATS (`aios.<tenant>.agent.…`). |
| `payload` | `jsonb` | NOT NULL | Envelope CloudEvents completo (RFC-0001 §5.2). |
| `published` | `boolean` | NOT NULL default false | Flag do relay. |
| `created_at` | `timestamptz` | NOT NULL | Enfileiramento. |

### 3.5 Relações (ASCII)

```
   Tenant(1) ───< AgentControlBlock(*) ───1 Quota
        │                 │  \                (scope=agent)
        │                 │   \───> memory_ptr → 010   context_ptr → 011
        │                 │        policy_ref  → 022
        │                 └───< SyscallRecord(*)  └───< OutboxMessage(*)
        └───1 Quota(scope=tenant)
   AgentControlBlock.parent_urn ──> AgentControlBlock (árvore de spawn)
```

---

## 4. Máquina de Estados Canônica do ACB

### 4.1 Estados (`AcbState`)

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Pending` | ACB criado; aguardando admissão do Scheduler. | não |
| `Admitted` | Scheduler aceitou; slot reservado; aguardando materialização. | não |
| `Running` | Runtime materializado; agente executando loop cognitivo. | não |
| `Suspended` | Agente pausado (voluntário ou por preempção); estado em RAM/quente. | não |
| `Hibernated` | Agente *cold*: estado em armazenamento durável (MinIO/PG); zero RAM. | não |
| `Terminating` | Encerramento em curso (drenagem + compensação). | não |
| `Terminated` | Encerrado; ACB retido para auditoria; recursos liberados. | **sim** |
| `Failed` | Falha irrecuperável na materialização/execução. | **sim** |

### 4.2 Transições, gatilhos e guardas

| # | De → Para | Gatilho (syscall/evento) | Guarda (DEVE ser verdadeira) |
|---|-----------|--------------------------|------------------------------|
| T-01 | ∅ → `Pending` | `spawn` | Capability `agent:spawn` concedida ∧ cota `agent_count` do tenant disponível. |
| T-02 | `Pending` → `Admitted` | resposta do Scheduler | Scheduler retornou slot ∧ admissão aprovada (cotas ok). |
| T-03 | `Pending` → `Failed` | timeout/rejeição do Scheduler | Admissão negada ou timeout > `spawn.admission_timeout`. |
| T-04 | `Admitted` → `Running` | confirmação do Lifecycle | Runtime materializado ∧ `runtime_ref` atribuído ∧ contexto restaurado. |
| T-05 | `Admitted` → `Failed` | falha de materialização | Runtime não iniciou dentro de `spawn.boot_timeout`. |
| T-06 | `Running` → `Suspended` | `suspend` / preempção | Capability `agent:suspend` ∧ agente em estado suspensível (sem seção crítica não-idempotente). |
| T-07 | `Suspended` → `Running` | `resume` | Capability `agent:resume` ∧ slot re-reservado no Scheduler. |
| T-08 | `Suspended` → `Hibernated` | política de ociosidade | `now - last_active_at > hibernation.idle_ttl`. |
| T-09 | `Hibernated` → `Admitted` | `resume` / demanda | Checkpoint válido ∧ Scheduler reserva slot (materialização sob demanda, alvo p99 < 250 ms). |
| T-10 | `Running`/`Suspended`/`Hibernated` → `Terminating` | `kill` | Capability `agent:kill` ∧ (chamador = owner ∨ role admin). |
| T-11 | `Terminating` → `Terminated` | drenagem concluída | Compensações executadas ∧ cotas liberadas ∧ evento `terminated` emitido. |
| T-12 | `Running` → `Failed` | falha do runtime | Heartbeat perdido > `runtime.heartbeat_deadline` ∧ retries esgotados. |
| T-13 | `Running`/`Suspended` → (mesmo) | `checkpoint` | Capability `agent:checkpoint`; snapshot durável concluído; atualiza `checkpoint_ref`. |

**Invariantes:** (I1) só há `runtime_ref` não-nulo em `Running`/`Suspended`.
(I2) todo estado terminal preenche `terminated_at` e libera cotas de agente.
(I3) transição só ocorre com `version` esperado (OCC); conflito → retry ou `AIOS-KERNEL-0009`.
(I4) toda transição emite exatamente um evento `aios.<tenant>.agent.lifecycle.<acao>` via Outbox.

### 4.3 Diagrama de estados (ASCII)

```
                    spawn (T-01)
        ∅ ─────────────────────────▶ ┌─────────┐
                                      │ Pending │
                                      └────┬────┘
                     admit (T-02)          │  reject/timeout (T-03)
             ┌────────────────────────────┤────────────────────────┐
             ▼                             │                        ▼
        ┌─────────┐   boot ok (T-04)       │                   ┌────────┐
        │Admitted │───────────────┐        │                   │ Failed │◀── boot fail (T-05)
        └────┬────┘               ▼        │                   └────────┘        ▲
             │             ┌───────────┐   │                                     │ runtime fail
   resume    │             │  Running  │───┼──── checkpoint (T-13, self) ───┐    │  (T-12)
   (T-09)    │             └──┬───┬────┘   │                                │    │
             │        suspend │   │ kill (T-10)                             ▼    │
             │         (T-06) │   └──────────────┐               ┌──────────────┴──┐
             │                ▼                  ▼               │   (self loop)   │
             │          ┌───────────┐      ┌────────────┐        └─────────────────┘
             └──────────│ Suspended │      │Terminating │
                idle    └──┬────┬───┘      └─────┬──────┘
                (T-08)     │    │ resume (T-07)  │ drain done (T-11)
                           ▼    └──────────▶Running     ▼
                     ┌───────────┐            ┌────────────┐
                     │Hibernated │─resume────▶│ Terminated │  (terminal)
                     └───────────┘  (T-09)    └────────────┘
```

---

## 5. Superfície de API (REST e gRPC)

> Autenticação/erros/idempotência/versionamento seguem RFC-0001 §5.4–§5.7. Pacote
> gRPC: `aios.kernel.v1`. Base REST: `/v1/kernel`. Toda mutação DEVE aceitar
> `Idempotency-Key` e os cabeçalhos de correlação (RFC-0001 §5.6).

### 5.1 Syscalls cognitivas → endpoints

| Verbo (syscall) | REST | gRPC (rpc) | Idempotente | Notas |
|-----------------|------|-----------|-------------|-------|
| `spawn` | `POST /v1/kernel/agents` | `Spawn` | sim (key) | Cria ACB → `Pending`. |
| `kill` | `DELETE /v1/kernel/agents/{urn}` | `Kill` | sim | → `Terminating`. |
| `suspend` | `POST /v1/kernel/agents/{urn}:suspend` | `Suspend` | sim | → `Suspended`. |
| `resume` | `POST /v1/kernel/agents/{urn}:resume` | `Resume` | sim | → `Running`. |
| `remember` | `POST /v1/kernel/agents/{urn}/memory` | `Remember` | sim (key) | Broker → `010-Memory`. |
| `recall` | `POST /v1/kernel/agents/{urn}/memory:query` | `Recall` | sim (leitura) | Broker → `010-Memory`. |
| `plan` | `POST /v1/kernel/agents/{urn}/plans` | `Plan` | sim (key) | Broker → `012-Planning`. |
| `invoke_tool` | `POST /v1/kernel/agents/{urn}/tools:invoke` | `InvokeTool` | sim (key) | Broker → `015-Tool`. |
| `route_model` | `POST /v1/kernel/agents/{urn}/inference:route` | `RouteModel` | sim (key) | Broker → `017-Model-Router`. |
| `get_quota` | `GET /v1/kernel/agents/{urn}/quota` | `GetQuota` | sim (leitura) | Saldo atual de cotas. |
| `checkpoint` | `POST /v1/kernel/agents/{urn}:checkpoint` | `Checkpoint` | sim (key) | Snapshot durável. |
| (admin) | `GET /v1/kernel/agents/{urn}` | `GetAcb` | sim (leitura) | Lê o ACB. |

### 5.2 Catálogo de códigos de erro (domínios reservados ao Kernel)

> Formato RFC-0001 §5.4: `AIOS-<DOMINIO>-<NNNN>`. Domínios reservados por este módulo:
> **`KERNEL`** (0001–0099), **`CAP`** (0001–0099, capability), **`QUOTA`** (0001–0099,
> compartilhado com `026`; o Kernel usa `AIOS-QUOTA-0001..0020`), **`SYSCALL`** (0001–0099).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-KERNEL-0001` | 404 | não | ACB/agente não encontrado. |
| `AIOS-KERNEL-0002` | 409 | não | Transição de estado inválida (viola FSM §4). |
| `AIOS-KERNEL-0003` | 503 | sim | Scheduler indisponível (admissão). |
| `AIOS-KERNEL-0004` | 504 | sim | Timeout de materialização (boot). |
| `AIOS-KERNEL-0005` | 502 | sim | Falha de broker de recurso (010/011/012/015/017). |
| `AIOS-KERNEL-0009` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-KERNEL-0010` | 400 | não | Payload/schema de syscall inválido. |
| `AIOS-CAP-0001` | 403 | não | Capability negada pelo PDP (*default deny*). |
| `AIOS-CAP-0002` | 403 | não | Tenant divergente do contexto autenticado. |
| `AIOS-CAP-0003` | 503 | sim | PDP indisponível e `fail-closed` ativo. |
| `AIOS-QUOTA-0001` | 429 | sim | Orçamento de tokens/custo excedido (hard). |
| `AIOS-QUOTA-0002` | 429 | sim | Rate-limit de syscalls excedido. |
| `AIOS-QUOTA-0003` | 429 | sim | Limite de ferramentas/inferências excedido. |
| `AIOS-SYSCALL-0001` | 400 | não | Verbo desconhecido ou versão de ABI não suportada. |
| `AIOS-SYSCALL-0002` | 409 | não | `Idempotency-Key` reusada com payload divergente. |

---

## 6. Catálogo de Eventos NATS

> Envelope CloudEvents e convenção de subjects: RFC-0001 §5.2–§5.3
> (`aios.<tenant>.<dominio>.<entidade>.<acao>`). `<dominio>` = `agent`. Entrega
> at-least-once via JetStream; consumidores DEVEM deduplicar por `event.id`.

### 6.1 Eventos emitidos (produtor: Kernel)

| Subject (`aios.<tenant>.…`) | `type` | Quando | Stream |
|-----------------------------|--------|--------|--------|
| `agent.lifecycle.spawned` | `aios.agent.lifecycle.spawned` | ACB criado (`Pending`). | KERNEL_LIFECYCLE (durável) |
| `agent.lifecycle.admitted` | `aios.agent.lifecycle.admitted` | Scheduler admitiu (`Admitted`). | KERNEL_LIFECYCLE |
| `agent.lifecycle.running` | `aios.agent.lifecycle.running` | Runtime materializado (`Running`). | KERNEL_LIFECYCLE |
| `agent.lifecycle.suspended` | `aios.agent.lifecycle.suspended` | Suspensão (`Suspended`). | KERNEL_LIFECYCLE |
| `agent.lifecycle.hibernated` | `aios.agent.lifecycle.hibernated` | Hibernação (`Hibernated`). | KERNEL_LIFECYCLE |
| `agent.lifecycle.resumed` | `aios.agent.lifecycle.resumed` | Retomada. | KERNEL_LIFECYCLE |
| `agent.lifecycle.terminated` | `aios.agent.lifecycle.terminated` | Encerramento (`Terminated`). | KERNEL_LIFECYCLE |
| `agent.lifecycle.failed` | `aios.agent.lifecycle.failed` | Falha irrecuperável (`Failed`). | KERNEL_LIFECYCLE |
| `agent.checkpoint.created` | `aios.agent.checkpoint.created` | `checkpoint` concluído. | KERNEL_LIFECYCLE |
| `agent.quota.exceeded` | `aios.agent.quota.exceeded` | Cota `hard` excedida. | KERNEL_QUOTA |
| `agent.quota.warning` | `aios.agent.quota.warning` | Cota `soft` em limiar (ex.: 80%). | KERNEL_QUOTA |
| `agent.syscall.denied` | `aios.agent.syscall.denied` | PEP negou (capability/tenant). | KERNEL_AUDIT |

### 6.2 Eventos consumidos

| Subject assinado | Produtor | Ação do Kernel |
|------------------|----------|----------------|
| `aios.<tenant>.scheduler.admission.decided` | `009-Scheduler` | Dirige T-02/T-03 do ACB. |
| `aios.<tenant>.scheduler.preemption.requested` | `009-Scheduler` | Dispara `suspend` (T-06). |
| `aios.<tenant>.lifecycle.runtime.started` | `008-Lifecycle` | Dirige T-04. |
| `aios.<tenant>.lifecycle.runtime.stopped` | `008-Lifecycle` | Dirige T-11/T-12. |
| `aios.<tenant>.policy.bundle.updated` | `022-Policy` | Invalida cache do CapabilityEnforcer. |
| `aios.<tenant>.cost.budget.updated` | `026-Cost-Optimizer` | Atualiza limites de `Quota`. |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito | Prioridade | Critério de aceite |
|----|-----------|-----------|--------------------|
| FR-001 | Expor os 11 verbos de syscall via REST e gRPC. | Must | Contrato OpenAPI+proto publicado; todos os verbos testados por contract test. |
| FR-002 | Toda syscall privilegiada DEVE passar pelo PEP e consultar o PDP antes de executar. | Must | Teste: syscall sem capability retorna `AIOS-CAP-0001`; auditoria registra `deny`. |
| FR-003 | Manter o ACB autoritativo com FSM da §4 e OCC. | Must | Transição inválida retorna `AIOS-KERNEL-0002`; conflito de versão retorna `AIOS-KERNEL-0009`. |
| FR-004 | Reservar/consumir/liberar cotas atomicamente por agente e tenant. | Must | Excesso `hard` retorna `AIOS-QUOTA-0001` e emite `agent.quota.exceeded`. |
| FR-005 | Coordenar `spawn` como saga (Scheduler→Lifecycle) com compensação. | Must | Falha de boot reverte reserva de slot e cota; ACB → `Failed`. |
| FR-006 | Garantir idempotência por `Idempotency-Key` (≥24h). | Must | Repetição retorna resultado idêntico; payload divergente → `AIOS-SYSCALL-0002`. |
| FR-007 | Emitir todos os eventos da §6 via Outbox transacional. | Must | Sem evento perdido em teste de crash entre commit e publish. |
| FR-008 | Brokerar `remember/recall/plan/invoke_tool/route_model` aos módulos donos. | Must | Cada broker aplica capability+cota antes de encaminhar. |
| FR-009 | Suportar hibernação/resume de agentes *cold* com checkpoint durável. | Should | Resume de `Hibernated` restaura ponteiros e retoma execução. |
| FR-010 | `get_quota` retorna saldo em tempo real (tokens/custo/tools/syscalls). | Must | Valor coerente com contadores Redis ±1 janela. |
| FR-011 | Rejeitar `tenant` divergente do contexto autenticado. | Must | Retorna `AIOS-CAP-0002`; evento `agent.syscall.denied`. |
| FR-012 | Registrar toda syscall em `syscall_log` e auditoria (`025`). | Must | 100% das ações privilegiadas auditadas. |

### 7.2 Requisitos Não-Funcionais (NFR) com SLO/SLI

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Desempenho (spawn) | p99 `spawn` ≤ **250 ms** (caminho quente, agente cold→running). | Histograma `aios_kernel_spawn_latency_ms`; load test. |
| NFR-002 | Desempenho (syscall de controle) | p99 `suspend/resume/kill/get_quota` ≤ **20 ms**. | `aios_kernel_syscall_latency_ms{verb}`. |
| NFR-003 | Desempenho (PEP) | p99 decisão de capability ≤ **8 ms** (com cache), ≤ 25 ms (cache miss). | `aios_kernel_pep_decision_ms`. |
| NFR-004 | Throughput | ≥ **20.000 syscalls/s** por réplica de Kernel (mix de controle). | Benchmark `Benchmark.md`. |
| NFR-005 | Disponibilidade | ≥ **99,95%** do serviço Kernel (control plane). | Uptime probe; error budget mensal. |
| NFR-006 | Escalabilidade | Suportar ≥ **10⁶** ACBs (majoritariamente `Hibernated`) com crescimento sub-linear de custo. | Teste de escala `Scalability.md`. |
| NFR-007 | Durabilidade de eventos | Perda de evento = **0** sob crash (Outbox + JetStream). | Chaos test kill entre commit/publish. |
| NFR-008 | Recuperação | RTO ≤ **15 min**, RPO ≤ **5 min** (estado de controle). | DR drill; ver §9. |
| NFR-009 | Consistência de cota | Overshoot de cota `hard` ≤ **1%** sob concorrência. | Teste concorrente com token-bucket. |
| NFR-010 | Segurança | 100% das ações privilegiadas via PEP; 0 bypass. | Pentest + auditoria de cobertura. |
| NFR-011 | Observabilidade | 100% das syscalls com trace OTel + correlação (`trace_id`,`tenant_id`). | Inspeção de spans; cobertura de campos. |
| NFR-012 | Idempotência | Repetições produzem efeito único em 100% das mutações. | Teste de replay/duplicação. |

---

## 8. Chaves de Configuração Principais

> Escopo: `global` (cluster), `tenant`, `agent`. Recarregável indica *hot reload*.
> Prefixo de env var: `AIOS_KERNEL_`.

| Chave | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|---------|-------|--------|--------------|-----------|
| `kernel.spawn.admission_timeout_ms` | 200 | 50–2000 | global/tenant | sim | Timeout de espera pela admissão do Scheduler. |
| `kernel.spawn.boot_timeout_ms` | 5000 | 500–30000 | global/tenant | sim | Timeout de materialização do runtime. |
| `kernel.pep.decision_cache_ttl_ms` | 3000 | 0–60000 | global | sim | TTL do cache de decisões do PEP. |
| `kernel.pep.fail_mode` | `closed` | {closed, open} | global/tenant | sim | Comportamento se PDP indisponível. `closed`=nega. |
| `kernel.quota.default_tokens` | 1000000 | ≥0 | tenant/agent | sim | Cota default de tokens por janela. |
| `kernel.quota.default_cost_usd` | 10.0 | ≥0 | tenant/agent | sim | Orçamento default por janela (sobreposto por `026`). |
| `kernel.quota.syscalls_per_sec` | 200 | 1–100000 | agent | sim | Rate-limit de syscalls por agente. |
| `kernel.quota.window` | `1h` | 1m–24h | tenant/agent | sim | Janela de reset de cotas. |
| `kernel.quota.enforcement` | `hard` | {hard, soft} | tenant/agent | sim | Modo de enforcement. |
| `kernel.hibernation.idle_ttl_ms` | 300000 | 0–86400000 | global/tenant | sim | Ociosidade até hibernar (`Suspended`→`Hibernated`). |
| `kernel.runtime.heartbeat_deadline_ms` | 15000 | 1000–120000 | global | sim | Deadline de heartbeat antes de `Failed`. |
| `kernel.acb.shard_count` | 64 | 1–4096 | global | **não** | Nº de shards (`hash mod N`). Reparticionar exige migração. |
| `kernel.outbox.relay_interval_ms` | 50 | 5–1000 | global | sim | Intervalo do relay do Outbox. |
| `kernel.idempotency.retention_h` | 48 | 24–720 | global | sim | Retenção de resultados idempotentes (≥24h por RFC-0001). |
| `kernel.acb.optimistic_retry_max` | 5 | 0–20 | global | sim | Retentativas em conflito de OCC. |
| `kernel.broker.circuit_breaker_threshold` | 0.5 | 0–1 | global | sim | Taxa de erro que abre o breaker por dependência. |

---

## 9. Modos de Falha e Recuperação

| Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---------------|----------|-----------|-------------|--------------------|
| PDP (022) indisponível | Timeout/CB no PolicyClient | Bulkhead do PEP | `fail_mode=closed`→nega (`AIOS-CAP-0003`); alerta | Retry idempotente após meia-abertura do CB |
| Scheduler (009) indisponível | Timeout na admissão | Saga aborta em `Pending` | ACB→`Failed` com compensação; cliente reenvia | `Idempotency-Key` evita ACB duplicado |
| Falha de boot do runtime | `boot_timeout` | Reverte reserva de slot | ACB→`Failed`; libera cota | Compensação idempotente |
| Crash do Kernel entre commit e publish | Relay do Outbox detecta `published=false` | Estado no PG intacto | Relay reenvia evento; consumidores deduplicam | Exactly-once efetivo (dedupe por `event.id`) |
| Conflito de concorrência (OCC) | `version` divergente | Transação isolada | Retry até `optimistic_retry_max`; senão `AIOS-KERNEL-0009` | Idempotente por natureza |
| Broker de recurso falho (010/011/…) | CB por dependência | Bulkhead isola a syscall | `AIOS-KERNEL-0005`; degradação graciosa | Retry idempotente com backoff |
| Overshoot de cota sob corrida | Reconciliação token-bucket | Contador atômico Redis (Lua) | Rejeita marginais; reconcilia saldo | Determinístico |
| Partição do NATS | Health do EventEmitter | Outbox retém eventos | Backlog drenado ao religar | Ordenado por stream; dedupe |
| Perda de Redis (contadores quentes) | Probe de saúde | Fallback a limites do PG | Recontagem conservadora (nega dúvidas) | Sem duplo débito |

**Metas de recuperação (control plane):** **RTO ≤ 15 min**, **RPO ≤ 5 min**
(alinhado a V-R10 da Visão e §2 da Arquitetura). Fonte da verdade em PostgreSQL HA
(streaming replication) + Outbox garante que nenhum evento aceito seja perdido.
Degradação graciosa: sob indisponibilidade de dependência, o Kernel prioriza
**negar com segurança** (default deny) em vez de permitir sem governança.

---

## 10. Escalabilidade e Concorrência

| Dimensão | Estratégia |
|----------|-----------|
| Particionamento | **Sharding determinístico** `shard = hash(tenant_id, agent_id) mod N` (`kernel.acb.shard_count`). Cada réplica do Kernel possui afinidade lógica a um subconjunto de shards; ACBs são roteados por shard para localidade de cache. |
| Estado quente vs. frio | ACB *hot* em **Redis**; fonte da verdade em **PostgreSQL** (particionado por `tenant_id` + `shard_key`). Agentes ociosos migram para `Hibernated` (estado em MinIO/PG), liberando RAM — habilita 10⁶⁺ ACBs com poucos ativos. |
| Concorrência de ACB | **OCC (optimistic concurrency)** por `version`; sem locks pessimistas no caminho quente. Conflitos raros resolvidos por retry idempotente. |
| Cotas concorrentes | **Token-bucket atômico em Redis** (script Lua) por `subject_urn`, evitando corrida; reconciliação periódica com PG. Overshoot ≤ 1% (NFR-009). |
| Backpressure | Admissão limitada pelo Scheduler; `syscalls_per_sec` por agente; JetStream com limites de stream; rejeição precoce (`429 AIOS-QUOTA-0002`) em vez de degradar todos (contenção de *noisy neighbor*). |
| Escala horizontal | Kernel Service é **stateless** (estado externalizado): réplicas escalam por CPU/fila. Sem afinidade obrigatória; afinidade de shard é otimização, não requisito de correção. |
| Caminho quente | `spawn`/controle sem I/O bloqueante síncrono desnecessário: PEP com cache, ACB em Redis, eventos assíncronos via Outbox. Alvo p99 spawn ≤ 250 ms / controle ≤ 20 ms. |
| Locks distribuídos | Somente onde estritamente necessário (ex.: transição única de FSM concorrente) via Redis lock com TTL curto; preferir OCC. |
| Rumo a milhões | Combinação: agentes majoritariamente *cold* + sharding + comunicação seletiva (limites de fan-out em `spawn` de sub-agentes) + cache de decisões. Ver `../001-Architecture/Architecture.md` §12. |

---

## 11. ADRs e RFCs a Propor

> **Faixa de ADR reservada a este módulo: `ADR-0060`..`ADR-0069`** (regra 006×10,
> evitando colisão entre módulos). Registrar em `../002-ADR/`.

| ADR | Título proposto |
|-----|-----------------|
| ADR-0060 | ABI de Syscalls Cognitivas: 11 verbos, versionamento e compatibilidade. |
| ADR-0061 | Modelo canônico do ACB e escolha de OCC (versão) sobre locks pessimistas. |
| ADR-0062 | Token-bucket atômico em Redis para enforcement de cotas com reconciliação em PostgreSQL. |
| ADR-0063 | PEP no Kernel vs. PDP no 022-Policy: fronteira, cache de decisões e `fail-closed`. |
| ADR-0064 | Coordenação de ciclo de vida como saga (Kernel↔Scheduler↔Lifecycle) com compensação. |
| ADR-0065 | Estratégia de sharding do ACB (`hash(tenant,agent) mod N`) e política de reparticionamento. |
| ADR-0066 | Outbox transacional + JetStream para entrega exactly-once efetiva de eventos de kernel. |
| ADR-0067 | Modelo de hibernação/cold-agent e contrato de checkpoint durável (MinIO). |
| ADR-0068 | Brokering governado de recursos (remember/recall/plan/invoke_tool/route_model) vs. acesso direto. |
| ADR-0069 | Domínios de código de erro reservados ao Kernel (`KERNEL`, `CAP`, `SYSCALL`) e faixa de `QUOTA`. |

| RFC | Título proposto | Relação |
|-----|-----------------|---------|
| RFC-0001 | Architecture Baseline & Core Contracts | **Baseline** (consumida, não redefinida). |
| RFC-0006 | Cognitive Syscall ABI (contrato dos 11 verbos, proto/OpenAPI). | A propor por este módulo. |
| RFC-0007 | ACB & Resource Quota Model (schema, FSM, enforcement). | A propor por este módulo. |

---

## 12. Decisões de Segurança

### 12.1 AuthN / AuthZ

- **AuthN**: identidade validada no **Gateway (YARP)** via OAuth2/OIDC (`021`); o Kernel
  **consome claims assinadas** (não emite/valida tokens). Comunicação interna via
  **mTLS** (RFC-0001 §6).
- **AuthZ**: o Kernel é **PEP**; toda syscall privilegiada consulta o **PDP** do
  `022-Policy` (RBAC + ABAC), com **default deny**. `capability_set` no ACB é apenas
  cache do último *grant*; a autoridade é o PDP.
- **Isolamento de tenant**: o Kernel **NÃO DEVE** aceitar `tenant` (em URN/subject/claim)
  divergente do contexto autenticado (RFC-0001 §6) → `AIOS-CAP-0002`.

### 12.2 Threat Model STRIDE (resumido)

| Ameaça (STRIDE) | Vetor no Kernel | Mitigação |
|-----------------|-----------------|-----------|
| **S**poofing | Agente forja identidade/tenant de outro. | mTLS interno + claims OIDC validadas no Gateway; `tenant` casado com contexto (`AIOS-CAP-0002`). |
| **T**ampering | Alteração do ACB ou de cotas. | OCC (`version`), RLS por `tenant_id`, escrita só via Kernel; auditoria imutável (`025`). |
| **R**epudiation | Negar ter executado uma syscall privilegiada. | `syscall_log` + auditoria imutável com `trace_id`, `agent_urn`, decisão do PEP. |
| **I**nformation disclosure | Vazamento entre tenants ou em erros. | RLS; namespace NATS por tenant; erros RFC-7807 sem PII (RFC-0001 §6); minimização de dados. |
| **D**enial of service | Flood de syscalls/spawn (noisy neighbor). | `syscalls_per_sec`, cotas `hard`, admissão do Scheduler, backpressure (`429`). |
| **E**levation of privilege | Executar syscall sem capability. | PEP→PDP default deny; sem caminho de bypass; cobertura auditada (NFR-010). |

### 12.3 LGPD / GDPR

- **Minimização**: payloads de eventos e logs do Kernel **NÃO DEVEM** conter PII do
  agente/usuário além do necessário; identificadores são URNs opacos (RFC-0001 §7).
- **Direito ao esquecimento**: `kill` + expurgo rastreável do ACB (retendo apenas
  metadados de auditoria exigidos por lei); operação de purge coordenada com `025`.
- **Base legal e retenção**: `syscall_log` e Outbox têm política de retenção
  configurável (`kernel.idempotency.retention_h` e retenção de streams JetStream),
  sem armazenar conteúdo de memória (isso pertence a `010`, que carrega base legal).
- **Segregação**: dados de tenants distintos são isolados por RLS e por conta NATS.

---

## 13. Referências

- Visão: `../000-Vision/Vision.md`
- Arquitetura: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Template de módulo: `../_templates/MODULE_TEMPLATE.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../022-Policy/`, `../009-Scheduler/`, `../008-Lifecycle/`,
  `../007-Agent-Runtime/`, `../010-Memory/`, `../011-Context/`, `../012-Planning/`,
  `../015-Tool-Manager/`, `../017-Model-Router/`, `../026-Cost-Optimizer/`,
  `../024-Observability/`, `../025-Audit/`, `../020-Communication/`.

*Fim do Design Brief interno do módulo 006-Kernel. Este documento governa a geração
dos 26 documentos obrigatórios; qualquer conflito entre um documento gerado e este
brief é um defeito do documento gerado.*
