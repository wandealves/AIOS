---
Documento: Design Brief (Fonte Única de Verdade Interna)
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0070, ADR-0071, ADR-0072, ADR-0073, ADR-0074, ADR-0075, ADR-0076, ADR-0077, ADR-0078, ADR-0079 (propostas por este módulo); ADR-0001, ADR-0002, ADR-0003 (globais)
RFCs relacionados: RFC-0001 (baseline), RFC-0070 (Runtime↔Supervisor Control Protocol — proposta), RFC-0071 (MCP Host Sandboxing Profile — proposta)
Depende de: 006-Kernel, 008-Agent-Lifecycle, 009-Scheduler, 010-Memory, 011-Context, 012-Planning, 015-Tool-Manager, 017-Model-Router, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit
---

# AIOS — Design Brief do Módulo 007 · Agent Runtime

> **Escopo deste documento.** Este brief é a **fonte única de verdade interna** do
> módulo `007-Agent-Runtime`. Todos os 26 documentos obrigatórios definidos em
> `../_templates/MODULE_TEMPLATE.md` DEVEM ser derivados deste brief e **NÃO
> PODEM** contradizê-lo. Termos reutilizam `../040-Glossary/Glossary.md`; os
> contratos centrais (URN, envelope de evento CloudEvents, envelope de erro
> RFC 7807, `Idempotency-Key`, correlação `traceparent`/`X-AIOS-Tenant`, subjects
> `aios.<tenant>.<dominio>.<entidade>.<acao>`) são os de
> `../003-RFC/RFC-0001-Architecture-Baseline.md` e **não são redefinidos** aqui,
> apenas referenciados.

As palavras-chave **DEVE**, **NÃO DEVE**, **DEVERIA**, **NÃO DEVERIA**, **PODE** e
**OPCIONAL** seguem RFC 2119 / RFC 8174.

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Posição na arquitetura

O Agent Runtime é o **plano de dados (data plane)** do AIOS, escrito em **Python**.
Ele executa o **loop cognitivo** de um agente isolado em **sandbox**, hospeda o
**MCP host** para ferramentas, invoca ferramentas exclusivamente via
`015-Tool-Manager` e modelos exclusivamente via `017-Model-Router`. Um processo
supervisor separado, o **Runtime Supervisor** (`.NET 10`, pertencente ao plano de
controle, descrito em `001-Architecture` e coordenado por `008-Agent-Lifecycle`),
gerencia o **pool** de runtimes (cria/monitora/mata), aplica cotas de nível de
pool e faz o *placement*. Este módulo especifica o **processo runtime Python** e o
**contrato** que ele expõe ao Supervisor; a implementação do Supervisor em .NET é
responsabilidade compartilhada com `008-Agent-Lifecycle`/`009-Scheduler` e está fora do
código deste módulo, embora o **protocolo** seja co-definido aqui (RFC-0070).

### 1.2 Responsabilidades (o que o módulo DEVE fazer)

| # | Responsabilidade |
|---|------------------|
| R-01 | Materializar (boot) um agente a partir de um **AgentSpec** entregue pelo Supervisor e conduzir seu ciclo de execução até `done`/`failed`/`suspended`. |
| R-02 | Executar o **loop cognitivo ReAct**: `perceber → pensar → agir(tool) → observar → repetir`, com replanejamento delegado a `012-Planning`. |
| R-03 | Hospedar o **MCP host** e expor as ferramentas permitidas ao raciocínio do agente, mediando toda invocação por `015-Tool-Manager`. |
| R-04 | Executar **todas** as chamadas de inferência (LLM) via `017-Model-Router` — **nunca** direto a um provedor. |
| R-05 | Recuperar/persistir memória via `010-Memory` e montar/compactar contexto via `011-Context`; **nunca** acessar PostgreSQL/pgvector/AGE diretamente para estado de controle. |
| R-06 | Aplicar **isolamento de sandbox** por agente: namespaces de processo, `seccomp-bpf`, `cgroups v2` (CPU/RAM/PIDs), filesystem read-only + escrita efêmera, e **negação de rede** salvo o egress explicitamente permitido pela política (`022-Policy`). |
| R-07 | Ser um **PEP local** (Policy Enforcement Point): toda ação privilegiada (tool, egress, syscall cognitiva) DEVE checar a *capability* concedida no boot e, quando exigido, consultar o PDP via Kernel/Policy. *Default deny*. |
| R-08 | Emitir **telemetria OTel** (traces/metrics/logs) e **eventos NATS/JetStream** de ciclo de vida e execução, com correlação `traceparent`/`tenant`, via **outbox** para atomicidade. |
| R-09 | Gerar **checkpoints** de estado do agente (working memory + posição no plano + cursor do loop) para permitir suspensão/hibernação (*cold agent*) e retomada com continuidade. |
| R-10 | Honrar **cotas locais** (tokens, custo USD, wall-clock, passos do loop, profundidade de recursão, fan-out de tools) e **abortar/suspender** ao esgotá-las, emitindo o erro canônico. |
| R-11 | Expor **health/liveness/readiness** e um canal de controle (**RuntimeControl**, gRPC + NATS) para o Supervisor comandar `boot/suspend/resume/kill/drain`. |
| R-12 | Garantir **idempotência** na aceitação de tarefas (`Idempotency-Key`) e **deduplicação** de eventos consumidos por `event.id`. |
| R-13 | Fazer **degradação graciosa** (fallback de modelo, redução de contexto, *circuit breaker* em tools/LLM) e **replanejamento** ao encontrar falhas recuperáveis. |
| R-14 | Aplicar **redação de PII** em payloads de evento/log conforme minimização de dados (LGPD/GDPR), delegando classificação de sensibilidade aos metadados de `010-Memory`. |

### 1.3 Não-responsabilidades (o que o módulo NÃO DEVE fazer)

| # | Não-responsabilidade | Dono correto |
|---|----------------------|--------------|
| N-01 | Decidir **onde**/**quando**/**com que prioridade** um agente executa (scheduling, admissão, preempção). | `009-Scheduler` |
| N-02 | Gerenciar o **pool** de runtimes (criar/monitorar/matar réplicas), autoscaling e cotas de pool. | Runtime Supervisor (`008-Agent-Lifecycle`) |
| N-03 | **Treinar/fine-tunar** modelos ou escolher provedor de LLM por custo/qualidade. | `017-Model-Router`, `026-Cost-Optimizer` |
| N-04 | **Registrar/versionar** ferramentas, resolver permissões globais ou implementar drivers de tool. | `015-Tool-Manager` |
| N-05 | Ser fonte da verdade de **memória**, **contexto**, **conhecimento** ou **auditoria** (apenas produtor/consumidor). | `010`, `011`, `018/019`, `025` |
| N-06 | Definir **políticas** RBAC/ABAC (é PEP, não PDP). | `022-Policy` |
| N-07 | Persistir o **event store** ou implementar CQRS/projeções de leitura globais. | serviços do control plane |
| N-08 | Expor API pública externa ao usuário final (é chamado pelo control plane; o gateway externo é o YARP). | `004-API`, API Gateway |
| N-09 | Orquestrar **workflows determinísticos/sagas** de múltiplos agentes. | `014-Workflow` |
| N-10 | Consolidar aprendizado/*reflexion* de longo prazo em novas políticas. | `023-Learning` |

---

## 2. Decomposição em Componentes Internos

### 2.1 Catálogo de componentes (PascalCase)

| Componente | Responsabilidade | Depende de (interno/externo) |
|------------|------------------|------------------------------|
| **RuntimeBootstrapper** | Inicializa o processo runtime, carrega o **AgentSpec**, valida *capabilities*, prepara sandbox e emite `boot→ready`. Alvo cold start ≤ 250 ms com pool quente. | SandboxManager, CapabilityEnforcer, SupervisorChannel |
| **SandboxManager** | Cria e mantém o isolamento: namespaces, `seccomp-bpf`, `cgroups v2`, FS efêmero, política de rede (egress allowlist). Aplica limites e mata em violação. | `021-Security` (perfis), `022-Policy` (egress) |
| **ExecutionStateMachine** | Máquina de estados canônica do agente (§4). Fonte da verdade **local** do estado; publica transições. | EventPublisher, TelemetryEmitter |
| **CognitiveLoopEngine** | Driver do loop ReAct: sequencia `Perception→Reasoning→Action→Observation`, controla passos/orçamento, dispara replanejamento e reflexão. | PerceptionModule, ReasoningModule, ActionExecutor, ObservationCollector, PlanningClient, QuotaGovernor |
| **PerceptionModule** | Monta o *percept* do passo: objetivo/goal atual, contexto recuperado, observações prévias, estado do plano. Chama Context/Memory. | ContextClient, MemoryClient |
| **ReasoningModule** | Transforma *percept* em *thought*/decisão de ação via inferência LLM; produz a próxima ação (tool call, resposta, replanejar, concluir). **Sempre** via Model Router. | ModelRouterClient, ContextClient |
| **ActionExecutor** | Executa a ação decidida: invoca tool via MCP host, produz resposta final ou aciona replanejamento. Aplica *circuit breaker*/timeout/retry. | McpHost, ToolInvoker, CapabilityEnforcer |
| **ObservationCollector** | Normaliza o resultado da ação (sucesso/erro/observação) em um **ReActStep**, atualiza working memory e alimenta o próximo *percept*. | MemoryClient, CheckpointManager |
| **McpHost** | Hospeda o servidor/host MCP dentro do sandbox; expõe o catálogo de ferramentas permitido, faz *handshake* e roteia chamadas MCP. | ToolInvoker, SandboxManager |
| **ToolInvoker** | Proxy de invocação de ferramenta; toda chamada passa por `015-Tool-Manager` (autorização, métricas, versão). Nunca chama a tool externamente por conta própria. | `015-Tool-Manager` (gRPC/NATS), CapabilityEnforcer |
| **ModelRouterClient** | Cliente de inferência; envia prompts/mensagens ao `017-Model-Router`, recebe modelo/endpoint escolhido, streaming e fallback. | `017-Model-Router` (gRPC) |
| **MemoryClient** | Recupera/persiste itens de memória (working/short/long/…); nunca toca PostgreSQL diretamente. | `010-Memory` (gRPC/NATS) |
| **ContextClient** | Monta janela de contexto, aplica compressão e cache semântico. | `011-Context` (gRPC) |
| **PlanningClient** | Solicita plano/decomposição e replanejamento; recebe passos e critérios de sucesso. | `012-Planning` (gRPC) |
| **CheckpointManager** | Serializa/deserializa o **Checkpoint** (working memory, cursor do loop, plano) para MinIO/PG via control plane; habilita suspend/resume (*cold agent*). | `010-Memory`, MinIO (via control plane), SupervisorChannel |
| **QuotaGovernor** | Contabiliza e impõe cotas locais (tokens, USD, wall-clock, passos, fan-out, profundidade). Sinaliza esgotamento e aciona suspensão/abort. | `026-Cost-Optimizer` (limites), Kernel (cotas) |
| **CapabilityEnforcer** | PEP local: valida *capability tokens* concedidos no boot e consulta o PDP quando a política exige decisão em tempo de ação. *Default deny*. | `022-Policy` (PDP via Kernel), `021-Security` |
| **EventPublisher** | Publica eventos de domínio no NATS/JetStream com envelope CloudEvents (RFC-0001) via **outbox** transacional local (SQLite efêmero/WAL do sandbox) para atomicidade at-least-once. | NATS/JetStream (`020`) |
| **TelemetryEmitter** | Emite traces/metrics/logs OTel com correlação `traceparent`/`tenant`/`agent`; exporta ao OTel Collector. | `024-Observability` |
| **SupervisorChannel** | Canal de controle com o Runtime Supervisor: recebe `boot/suspend/resume/kill/drain`, reporta health/heartbeat e estado. gRPC (comando) + NATS (heartbeat/eventos). | Runtime Supervisor, `008-Agent-Lifecycle` |
| **HealthProbe** | Endpoints/handlers de liveness, readiness e *startup*; watchdog do loop (detecta *stuck* e deadlock). | ExecutionStateMachine, SupervisorChannel |

### 2.2 Diagrama de componentes (ASCII)

```
                     ┌───────────────────────────── RUNTIME SUPERVISOR (.NET 10) ─────────────────────┐
                     │      cria/monitora/mata pool · placement · cotas de pool · drain               │
                     └───────────────┬───────────────────────────────────────────────┬───────────────┘
                       gRPC (comando) │                                    NATS (heartbeat/eventos) │
   ╔═══════════════════════════════════▼════════════ AGENT RUNTIME (Python · processo por agente) ═══▼══════════╗
   ║  ┌───────────────────┐   ┌────────────────────┐   ┌──────────────────────┐                                 ║
   ║  │ SupervisorChannel │──▶│ RuntimeBootstrapper │──▶│  SandboxManager      │  seccomp·cgroups·netns·FS ro    ║
   ║  └─────────┬─────────┘   └─────────┬──────────┘   └──────────┬───────────┘                                 ║
   ║            │ heartbeat/health        │ AgentSpec, capabilities  │ isola tudo abaixo                          ║
   ║  ┌─────────▼─────────┐   ┌──────────▼───────────────────────────▼──────────────────────────────────────┐   ║
   ║  │   HealthProbe     │   │              ExecutionStateMachine (§4)  ── estado canônico local            │   ║
   ║  └───────────────────┘   └──────────┬───────────────────────────────────────────────────┬─────────────┘   ║
   ║                                      │ conduz                                              │ transições       ║
   ║                          ┌───────────▼──────────── CognitiveLoopEngine (ReAct) ───────────▼───────────┐     ║
   ║                          │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │     ║
   ║                          │  │ Perception   │─▶│ Reasoning    │─▶│ ActionExecutor│─▶│ Observation      │ │     ║
   ║                          │  │ Module       │  │ Module       │  │              │  │ Collector        │ │     ║
   ║                          │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘ │     ║
   ║                          │         │ percept          │ thought/act     │ tool/answer      │ observe    │     ║
   ║                          └─────────┼──────────────────┼─────────────────┼──────────────────┼───────────┘     ║
   ║        ┌─────────────────┐         │                  │                 │                  │                 ║
   ║        │ QuotaGovernor   │◀── orçamento/passos ────────┴────────┬────────┴──────────────────┘                 ║
   ║        └─────────────────┘                                      │                                             ║
   ║  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────▼──┐  ┌──────────────┐  ┌──────────────────┐   ║
   ║  │ ContextClient    │  │ MemoryClient     │  │  McpHost           │  │ PlanningClient│  │ CapabilityEnforcer│  ║
   ║  │  → 011-Context   │  │  → 010-Memory    │  │   ┌──────────────┐ │  │ → 012-Planning│  │  PEP (→022-Policy)│  ║
   ║  └──────────────────┘  └──────────────────┘  │   │ ToolInvoker  │─┼──┼─▶ 015-Tool-Mgr└──────────────────┘   ║
   ║  ┌──────────────────┐  ┌──────────────────┐  │   └──────────────┘ │  └──────────────┘                         ║
   ║  │ ModelRouterClient│  │ CheckpointManager│  └────────────────────┘                                          ║
   ║  │  → 017-Model-Rtr │  │  → MinIO/PG(cp)  │                                                                  ║
   ║  └──────────────────┘  └──────────────────┘                                                                  ║
   ║  ┌──────────────────┐  ┌──────────────────┐                                                                  ║
   ║  │ EventPublisher   │  │ TelemetryEmitter │   outbox → NATS/JetStream (020) · OTel → Collector (024)         ║
   ║  └────────┬─────────┘  └────────┬─────────┘                                                                  ║
   ╚═══════════│═════════════════════│════════════════════════════════════════════════════════════════════════════╝
              ▼                     ▼
    aios.<tenant>.agent.runtime.*   OTel Collector (traces/metrics/logs)
    aios.<tenant>.task.execution.*
```

**Regras de fronteira (invioláveis, herdadas de `001-Architecture` §6):**
- Nenhum componente fala com LLM exceto **ModelRouterClient → 017**.
- Nenhum componente executa ferramenta externa exceto **ToolInvoker → 015**.
- Nenhum componente acessa PostgreSQL/pgvector/AGE diretamente; estado de controle
  trafega por gRPC/NATS aos serviços do control plane.
- Toda ação privilegiada passa pelo **CapabilityEnforcer** (PEP) — *default deny*.

---

## 3. Modelo de Dados / Entidades Canônico

### 3.1 Identidade

Recursos multi-tenant seguem o URN de RFC-0001 §5.1:
`urn:aios:<tenant>:<tipo>:<id>`, com `<id>` = **ULID**. Este módulo consome os tipos
`agent`, `task`, `tool`, `model`, `plan` e **propõe** (via ADR-0070) o registro do
tipo `runtime` no registro de tipos de recurso (`004-API/Resources.md`). Enquanto o
registro não é aceito, o identificador de instância de runtime é um ULID interno
(`runtime_id`) referenciado por relação, **não** como URN público. Toda entidade
persistida ou publicada DEVE carregar `tenant_id` (fronteira de isolamento; RLS no
control plane).

### 3.2 Entidades

#### RuntimeInstance — instância de runtime no pool (efêmera)

| Campo | Tipo | Chave/Constraint | Notas |
|-------|------|------------------|-------|
| `runtime_id` | ULID | PK | Identidade da instância do processo runtime. |
| `tenant_id` | slug (a-z0-9-) | NOT NULL, idx | Fronteira de isolamento. |
| `pool_id` | string | NOT NULL, idx | Pool ao qual pertence (Supervisor). |
| `agent_urn` | URN(agent) | NULL até atribuição | Agente atualmente materializado. |
| `shard` | int | NOT NULL | `hash(tenant_id, agent_id) mod N`. |
| `state` | enum(RuntimeState) | NOT NULL | `warming\|idle\|busy\|draining\|dead`. |
| `sandbox_profile_id` | string | FK→SandboxProfile | Perfil aplicado. |
| `node_id` | string | NOT NULL | Nó/host físico. |
| `started_at` | timestamptz | NOT NULL | Boot do processo. |
| `last_heartbeat_at` | timestamptz | NOT NULL | Watchdog. |
| `warm` | bool | default false | Faz parte do pool quente (cold start ≤250ms). |

#### AgentSession — execução de um agente materializado

| Campo | Tipo | Chave/Constraint | Notas |
|-------|------|------------------|-------|
| `session_id` | ULID | PK | Uma materialização do agente. |
| `agent_urn` | URN(agent) | NOT NULL, idx | Agente. |
| `task_urn` | URN(task) | NOT NULL, idx | Tarefa sendo executada. |
| `tenant_id` | slug | NOT NULL, idx (RLS) | Isolamento. |
| `runtime_id` | ULID | FK→RuntimeInstance | Onde executa. |
| `state` | enum(AgentExecState) | NOT NULL | Estado canônico (§4). |
| `plan_urn` | URN(plan) | NULL | Plano ativo (012). |
| `step_cursor` | int | NOT NULL default 0 | Passo atual do loop ReAct. |
| `budget` | ResourceBudget (jsonb) | NOT NULL | Cotas locais (tokens/USD/wall/steps). |
| `consumed` | ResourceUsage (jsonb) | NOT NULL | Consumo acumulado. |
| `idempotency_key` | string | UNIQUE(tenant,task) | RFC-0001 §5.5. |
| `traceparent` | string | NOT NULL | W3C Trace Context. |
| `created_at` | timestamptz | NOT NULL | — |
| `updated_at` | timestamptz | NOT NULL | — |
| `terminal_reason` | string | NULL | Preenchido em `done`/`failed`. |

#### ReActStep — passo do loop cognitivo (event-sourced, append-only)

| Campo | Tipo | Chave/Constraint | Notas |
|-------|------|------------------|-------|
| `step_id` | ULID | PK | Ordenável por tempo. |
| `session_id` | ULID | FK→AgentSession, idx | — |
| `tenant_id` | slug | NOT NULL (RLS) | — |
| `index` | int | NOT NULL | Ordem no loop. |
| `phase` | enum | NOT NULL | `perceive\|think\|act\|observe\|reflect`. |
| `percept_ref` | string | NULL | Ponteiro para contexto/percept. |
| `thought` | text | NULL | Raciocínio (redigido p/ PII). |
| `action_type` | enum | NULL | `tool_call\|final_answer\|replan\|noop`. |
| `tool_invocation_id` | ULID | FK→ToolInvocation NULL | Se `tool_call`. |
| `model_call_id` | ULID | FK→ModelCall NULL | Se `think`. |
| `observation` | jsonb | NULL | Resultado normalizado. |
| `status` | enum | NOT NULL | `ok\|error\|timeout\|denied`. |
| `latency_ms` | int | NOT NULL | Duração do passo. |
| `created_at` | timestamptz | NOT NULL | — |

#### ToolInvocation — chamada de ferramenta via 015

| Campo | Tipo | Chave/Constraint | Notas |
|-------|------|------------------|-------|
| `tool_invocation_id` | ULID | PK | — |
| `session_id` | ULID | FK, idx | — |
| `tenant_id` | slug | NOT NULL (RLS) | — |
| `tool_urn` | URN(tool) | NOT NULL | Ferramenta. |
| `capability_id` | string | NOT NULL | *Capability* validada (PEP). |
| `arguments_ref` | string | NOT NULL | Args (armazenados/redigidos). |
| `result_status` | enum | NOT NULL | `ok\|error\|timeout\|denied\|circuit_open`. |
| `retries` | int | default 0 | Tentativas. |
| `cost_usd` | numeric | default 0 | Custo atribuído. |
| `latency_ms` | int | NOT NULL | — |
| `idempotency_key` | string | NOT NULL | Dedup na invocação. |
| `created_at` | timestamptz | NOT NULL | — |

#### ModelCall — chamada de inferência via 017

| Campo | Tipo | Chave/Constraint | Notas |
|-------|------|------------------|-------|
| `model_call_id` | ULID | PK | — |
| `session_id` | ULID | FK, idx | — |
| `tenant_id` | slug | NOT NULL (RLS) | — |
| `requested_model` | string | NULL | Preferência (opcional). |
| `resolved_model_urn` | URN(model) | NOT NULL | Escolhido pelo Router. |
| `prompt_tokens` | int | NOT NULL | — |
| `completion_tokens` | int | NOT NULL | — |
| `cost_usd` | numeric | NOT NULL | — |
| `status` | enum | NOT NULL | `ok\|error\|timeout\|fallback`. |
| `latency_ms` | int | NOT NULL | — |
| `created_at` | timestamptz | NOT NULL | — |

#### Checkpoint — snapshot para suspend/resume (*cold agent*)

| Campo | Tipo | Chave/Constraint | Notas |
|-------|------|------------------|-------|
| `checkpoint_id` | ULID | PK | — |
| `session_id` | ULID | FK, idx | — |
| `tenant_id` | slug | NOT NULL (RLS) | — |
| `step_cursor` | int | NOT NULL | Retomada. |
| `working_memory_ref` | string(MinIO) | NOT NULL | Blob de estado. |
| `plan_snapshot_ref` | string | NULL | Plano congelado. |
| `size_bytes` | bigint | NOT NULL | — |
| `checksum` | string(sha256) | NOT NULL | Integridade. |
| `created_at` | timestamptz | NOT NULL | — |

#### SandboxProfile — perfil de isolamento

| Campo | Tipo | Chave/Constraint | Notas |
|-------|------|------------------|-------|
| `sandbox_profile_id` | string | PK | Ex.: `default`, `net-restricted`, `no-net`. |
| `tenant_id` | slug | NULL(global) / NOT NULL(custom) | Perfis por tenant. |
| `seccomp_profile` | jsonb | NOT NULL | Lista de syscalls permitidas. |
| `cpu_millicores` | int | NOT NULL | Limite `cgroups`. |
| `mem_mib` | int | NOT NULL | Limite `cgroups`. |
| `pids_max` | int | NOT NULL | Limite de PIDs. |
| `egress_allowlist` | jsonb | NOT NULL | Hosts/portas permitidos (default vazio). |
| `fs_writable_mib` | int | NOT NULL | tmpfs efêmero. |

### 3.3 Relações (ASCII)

```
 Tenant 1───* RuntimeInstance 1───* AgentSession 1───* ReActStep
                                        │  1                  │*
                                        ├──* ToolInvocation ◀─┘ (0..1 por step)
                                        ├──* ModelCall      ◀── (0..1 por step think)
                                        └──* Checkpoint
 SandboxProfile 1───* RuntimeInstance
```

Fonte da verdade **durável** (AgentSession/ReActStep/ToolInvocation/ModelCall/
Checkpoint) reside no control plane (PostgreSQL, com RLS por `tenant_id`) e é
escrita **via serviços** (`010`, `025`) e projeções event-sourced; o runtime mantém
apenas cópias **efêmeras** em sandbox (SQLite/WAL local para o outbox e cursor).

---

## 4. Máquinas de Estado Canônicas

### 4.1 Estado de execução do agente (AgentExecState)

Estados canônicos (semente do módulo): `boot`, `ready`, `thinking`, `acting`,
`waiting-io`, `reflecting`, `done`, `failed`. Este módulo adiciona os estados de
hibernação necessários ao *cold agent*: `suspending`, `suspended`, `resuming`,
`terminating`. `done`, `failed` são **terminais**; `suspended` é **quiescente**
(retomável).

```
        ┌────────┐  spec ok / sandbox pronto           ┌──────────┐
        │  boot  │────────────────────────────────────▶│  ready   │
        └───┬────┘  [G1: capabilities válidas]          └────┬─────┘
            │ falha boot [G0]                                 │ inicia passo
            ▼                                                 ▼
        ┌────────┐                                       ┌──────────┐  precisa tool
        │ failed │◀───── erro não recuperável ──────────│ thinking │──────────────┐
        └────────┘   [G7]                        ┌──────▶└────┬─────┘              │
            ▲                                     │           │ resposta final     ▼
            │ erro fatal                          │ observa   │ [G5]         ┌──────────┐
            │                                 ┌───┴────┐      │              │  acting  │
            │                  ┌──────────────│reflect.│◀─────┼──────────────│          │
            │                  │ replaneja    └────────┘ falha│ recuperável  └────┬─────┘
            │                  ▼  [G6]                        │ [G4]              │ I/O tool/LLM
       ┌─────────┐        ┌──────────┐  plano novo           ▼                   ▼
       │  done   │◀───────│ thinking │◀─── continua ───┐  ┌──────────┐      ┌──────────┐
       └─────────┘  [G5b] └──────────┘                 └──│waiting-io│◀─────│  acting  │
            ▲ objetivo cumprido                            └────┬─────┘ resp └──────────┘
            │                                                    │ resultado chega
            │            ┌───────────┐  sinal suspend            ▼
            └────────────│ suspended │◀── suspending ◀──── (qualquer estado ativo)
                         └─────┬─────┘   [G8: checkpoint ok]
                    resume     │
                   [G9] ┌──────▼─────┐
                        │  resuming  │──── restaura checkpoint ──▶ (estado salvo)
                        └────────────┘
        (kill/drain de qualquer estado) ──▶ terminating ──▶ done|failed
```

### 4.2 Tabela de transições e guardas

| # | De | Para | Gatilho | Guarda |
|---|----|------|---------|--------|
| T1 | boot | ready | Sandbox pronto + AgentSpec carregado | **G1**: *capabilities* válidas; cotas alocadas; MCP host ativo |
| T2 | boot | failed | Erro de inicialização | **G0**: falha de sandbox/spec/capabilities |
| T3 | ready | thinking | Início de passo do loop | orçamento disponível (QuotaGovernor) |
| T4 | thinking | acting | Decisão = `tool_call` | **G2**: *capability* da tool concedida (PEP) |
| T5 | thinking | done | Decisão = `final_answer` | **G5**: critério de sucesso do goal satisfeito |
| T6 | thinking | reflecting | Decisão = `replan` ou falha detectada | **G4**: erro recuperável / deriva de objetivo |
| T7 | acting | waiting-io | Tool/LLM em andamento (assíncrono) | invocação despachada, aguardando resultado |
| T8 | waiting-io | observing→thinking | Resultado chega | resultado normalizado por ObservationCollector |
| T9 | waiting-io | failed | Timeout/erro fatal da ação | **G7**: retries esgotados e não recuperável |
| T10 | reflecting | thinking | Novo plano de `012-Planning` | **G6**: plano válido recebido |
| T11 | reflecting | failed | Replanejamento impossível | **G7**: sem plano viável / orçamento esgotado |
| T12 | qualquer ativo | suspending | Sinal `suspend`/preempção/cota | comando do Supervisor ou QuotaGovernor |
| T13 | suspending | suspended | Checkpoint concluído | **G8**: checkpoint íntegro (checksum ok) |
| T14 | suspended | resuming | Sinal `resume` | **G9**: slot disponível + checkpoint recuperável |
| T15 | resuming | (estado salvo) | Restauração completa | working memory + cursor restaurados |
| T16 | qualquer | terminating | `kill`/`drain`/cota fatal | comando do Supervisor ou violação de sandbox |
| T17 | terminating | done/failed | Encerramento limpo/sujo | flush do outbox + liberação de recursos |

**Invariantes:** (i) toda transição publica um evento (§6) e emite um span OTel;
(ii) `consumed ≤ budget` DEVE ser mantido a cada passo, sob pena de transição
forçada a `suspending`/`terminating`; (iii) de `suspended` só se sai por `resume`
ou `terminating`; (iv) estados terminais NÃO transicionam.

### 4.3 Estado da instância de runtime (RuntimeState)

```
 warming ──ready──▶ idle ──atribui agente──▶ busy ──libera──▶ idle
    │                 │                         │ drain          │ drain
    │                 └──────────── draining ◀──┴────────────────┘
    └──erro──▶ dead ◀──── heartbeat perdido / violação de sandbox ────
```

| De | Para | Gatilho | Guarda |
|----|------|---------|--------|
| warming | idle | Processo pronto, entra no pool quente | health OK |
| idle | busy | Supervisor atribui AgentSession | slot livre |
| busy | idle | Sessão terminou (done/failed/suspended) | recursos liberados |
| idle/busy | draining | Comando `drain` (rollout/scale-down) | — |
| draining | dead | Sessões drenadas | sem sessões ativas |
| qualquer | dead | Heartbeat perdido/violação sandbox | watchdog |

---

## 5. Superfície de API

O Agent Runtime é um serviço **interno** do plano de dados: é comandado pelo
Runtime Supervisor e observa/produz via NATS. Sua superfície **gRPC** é o contrato
primário (RFC-0070); a superfície **REST** é restrita a *health*/introspecção e é
exposta apenas na rede de controle (nunca ao usuário final; o gateway externo é o
YARP, §N-08). Pacote gRPC versionado: `aios.runtime.v1`. Erros seguem RFC-0001 §5.4
com domínio **`RUNTIME`** (`AIOS-RUNTIME-<NNNN>`).

### 5.1 gRPC — `aios.runtime.v1.RuntimeControl` (Supervisor → Runtime)

| Método | Descrição | Idempotência |
|--------|-----------|--------------|
| `Boot(BootRequest) → BootReply` | Materializa agente a partir do AgentSpec; retorna `runtime_id`/`session_id`. | `Idempotency-Key` obrigatório |
| `Suspend(SuspendRequest) → SuspendReply` | Gera checkpoint e leva a `suspended`. | idempotente por `session_id` |
| `Resume(ResumeRequest) → ResumeReply` | Restaura de checkpoint. | idempotente por `checkpoint_id` |
| `Kill(KillRequest) → KillReply` | Encerramento forçado (`terminating`). | idempotente |
| `Drain(DrainRequest) → DrainReply` | Para de aceitar; drena sessões. | idempotente |
| `GetStatus(StatusRequest) → RuntimeStatus` | Estado/cotas/heartbeat. | leitura |

### 5.2 gRPC — `aios.runtime.v1.AgentExecution` (Control plane → Runtime)

| Método | Descrição | Idempotência |
|--------|-----------|--------------|
| `SubmitTask(TaskRequest) → stream ExecutionEvent` | Submete tarefa; devolve stream de passos ReAct/resultado. | `Idempotency-Key` obrigatório |
| `CancelTask(CancelRequest) → CancelReply` | Cancela tarefa em andamento. | idempotente por `task_urn` |
| `StreamSteps(SessionRef) → stream ReActStep` | Observa passos do loop (para o WebConsole via control plane). | leitura |

### 5.3 REST (rede de controle) — `/v1/runtime`

| Método | Recurso | Descrição |
|--------|---------|-----------|
| `GET` | `/v1/runtime/healthz` | Liveness. |
| `GET` | `/v1/runtime/readyz` | Readiness (pool quente ok). |
| `GET` | `/v1/runtime/{runtime_id}/status` | Status/cotas. |
| `GET` | `/v1/runtime/sessions/{session_id}` | Snapshot da sessão (RLS por `tenant_id`). |

### 5.4 Catálogo de códigos de erro (domínio `RUNTIME`, faixa reservada 0001–0099)

| Código | HTTP | Retriable | Significado |
|--------|------|-----------|-------------|
| `AIOS-RUNTIME-0001` | 503 | true | Pool sem slot quente disponível (cold start indisponível). |
| `AIOS-RUNTIME-0002` | 500 | false | Falha de boot do sandbox (seccomp/cgroups/netns). |
| `AIOS-RUNTIME-0003` | 403 | false | *Capability* ausente/negada (PEP *default deny*). |
| `AIOS-RUNTIME-0004` | 429 | true | Cota local esgotada (tokens/USD/wall-clock/passos). |
| `AIOS-RUNTIME-0005` | 504 | true | Timeout de invocação de tool (via 015). |
| `AIOS-RUNTIME-0006` | 502 | true | Falha do Model Router (inferência). |
| `AIOS-RUNTIME-0007` | 409 | false | Violação de idempotência (chave já usada com payload divergente). |
| `AIOS-RUNTIME-0008` | 500 | false | Loop travado / watchdog disparado (deadlock/*stuck*). |
| `AIOS-RUNTIME-0009` | 422 | false | AgentSpec inválido/incompatível. |
| `AIOS-RUNTIME-0010` | 500 | true | Falha ao gerar/restaurar checkpoint (integridade). |
| `AIOS-RUNTIME-0011` | 403 | false | Violação de política de rede (egress fora da allowlist). |
| `AIOS-RUNTIME-0012` | 409 | false | Máximo de passos/profundidade de recursão do loop excedido. |
| `AIOS-RUNTIME-0013` | 503 | true | Runtime em `draining`; não aceita novas tarefas. |
| `AIOS-RUNTIME-0014` | 400 | false | Replanejamento impossível (sem plano viável de 012). |
| `AIOS-RUNTIME-0015` | 500 | false | Violação de sandbox detectada (syscall proibida) — sessão morta. |

gRPC mapeia `status` → `google.rpc.Status` + `ErrorInfo` com o mesmo `code`
(RFC-0001 §5.4). `detail` NÃO DEVE vazar PII (RFC-0001 §6).

---

## 6. Catálogo de Eventos NATS

Subjects seguem RFC-0001 §5.3: `aios.<tenant>.<dominio>.<entidade>.<acao>`.
Envelope CloudEvents de RFC-0001 §5.2. Entrega **at-least-once**; consumidores
DEVEM deduplicar por `event.id`. Domínios usados: `agent` e `task`.

### 6.1 Emitidos (produtor: Agent Runtime)

| Subject | `type` | Quando | `data` (resumo) |
|---------|--------|--------|-----------------|
| `aios.<t>.agent.runtime.booted` | `aios.agent.runtime.booted` | boot→ready | `runtime_id`, `session_id`, `agent_urn`, `cold_start_ms` |
| `aios.<t>.agent.runtime.suspended` | `aios.agent.runtime.suspended` | →suspended | `session_id`, `checkpoint_id`, `step_cursor` |
| `aios.<t>.agent.runtime.resumed` | `aios.agent.runtime.resumed` | resuming→ativo | `session_id`, `checkpoint_id` |
| `aios.<t>.agent.runtime.terminated` | `aios.agent.runtime.terminated` | terminating→done/failed | `session_id`, `reason`, `terminal_state` |
| `aios.<t>.agent.runtime.heartbeat` | `aios.agent.runtime.heartbeat` | periódico | `runtime_id`, `state`, `consumed`, `budget` |
| `aios.<t>.task.execution.started` | `aios.task.execution.started` | ready→thinking (1º passo) | `task_urn`, `session_id`, `plan_urn` |
| `aios.<t>.task.execution.step` | `aios.task.execution.step` | cada ReActStep | `step_id`, `phase`, `status`, `latency_ms` |
| `aios.<t>.task.execution.completed` | `aios.task.execution.completed` | →done | `task_urn`, `status=SUCCEEDED`, `costUsd`, `tokens` |
| `aios.<t>.task.execution.failed` | `aios.task.execution.failed` | →failed | `task_urn`, `code`, `reason`, `retriable` |
| `aios.<t>.agent.runtime.quota_exceeded` | `aios.agent.runtime.quota_exceeded` | QuotaGovernor esgota cota | `session_id`, `resource`, `limit`, `consumed` |
| `aios.<t>.agent.runtime.sandbox_violation` | `aios.agent.runtime.sandbox_violation` | syscall/egress proibido | `session_id`, `violation_type` (auditoria 025) |

### 6.2 Consumidos (consumidor: Agent Runtime)

| Subject | Origem | Ação no runtime |
|---------|--------|-----------------|
| `aios.<t>.agent.lifecycle.spawn_requested` | Scheduler/Lifecycle (009/008) | Aciona `Boot`. |
| `aios.<t>.agent.lifecycle.suspend_requested` | Scheduler (preempção) | Aciona `Suspend`. |
| `aios.<t>.agent.lifecycle.resume_requested` | Scheduler/Lifecycle | Aciona `Resume`. |
| `aios.<t>.agent.lifecycle.kill_requested` | Kernel/Lifecycle | Aciona `Kill`. |
| `aios.<t>.policy.decision.updated` | Policy (022) | Invalida cache de *capabilities* do PEP local. |
| `aios.<t>.tool.registry.updated` | Tool Manager (015) | Recarrega catálogo do MCP host. |

**Outbox:** `EventPublisher` grava o evento em outbox local (SQLite/WAL efêmero) na
mesma transação que muda o estado, e publica de forma assíncrona com retry —
garantindo atomicidade estado↔evento (padrão Outbox de `001-Architecture` §13).

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito | Prioridade (MoSCoW) | Critério de aceite |
|----|-----------|---------------------|--------------------|
| FR-001 | Materializar agente a partir de AgentSpec e conduzir até estado terminal. | Must | `Boot` cria sessão e emite `agent.runtime.booted`; estado atinge `done`/`failed`. |
| FR-002 | Executar loop ReAct `perceber→pensar→agir→observar→repetir`. | Must | Cada iteração gera um `ReActStep`; evento `task.execution.step`. |
| FR-003 | Toda inferência via Model Router (017); nunca direta. | Must | Teste de contrato: zero chamadas a provedor sem passar por 017. |
| FR-004 | Toda tool via Tool Manager (015)/MCP host. | Must | Teste de contrato: zero invocações externas fora de 015. |
| FR-005 | Recuperar/persistir memória (010) e montar contexto (011). | Must | `PerceptionModule` popula percept a partir de 010/011. |
| FR-006 | Replanejar via Planning (012) em falha/deriva. | Must | Transição `reflecting→thinking` com `plan_urn` novo. |
| FR-007 | Isolar cada agente em sandbox (seccomp/cgroups/netns/FS/rede). | Must | Syscall proibida → `sandbox_violation` + kill. |
| FR-008 | Aplicar PEP local *default deny* para tools/egress/syscalls. | Must | Ação sem capability → `AIOS-RUNTIME-0003`. |
| FR-009 | Suspender/retomar agente via checkpoint (*cold agent*). | Must | `Suspend`→`suspended`; `Resume` restaura cursor e working memory. |
| FR-010 | Honrar cotas locais e abortar/suspender ao esgotar. | Must | `AIOS-RUNTIME-0004`/`quota_exceeded`. |
| FR-011 | Emitir eventos NATS + telemetria OTel correlacionada. | Must | Todos os subjects de §6 emitidos com envelope RFC-0001. |
| FR-012 | Idempotência de submissão de tarefa (`Idempotency-Key`). | Must | Repetição retorna mesmo resultado por ≥24h. |
| FR-013 | Streaming de passos/resultados ao control plane (SSE/gRPC stream). | Should | `StreamSteps`/`SubmitTask` emitem stream ordenado. |
| FR-014 | *Circuit breaker*/retry/backoff em tools e LLM. | Should | Falhas transitórias reintentadas; `circuit_open` reportado. |
| FR-015 | Redação de PII em eventos/logs. | Must | PII não aparece em payloads; conforme metadados de 010. |
| FR-016 | Watchdog anti-*stuck*/deadlock do loop. | Should | Loop travado → `AIOS-RUNTIME-0008` + recuperação/kill. |
| FR-017 | Limitar passos/profundidade/fan-out do loop. | Must | Excedente → `AIOS-RUNTIME-0012`. |
| FR-018 | Reportar heartbeat/health ao Supervisor. | Must | `agent.runtime.heartbeat` e `readyz`. |

### 7.2 Requisitos Não-Funcionais (NFR) com SLO/SLI

| ID | Atributo | Meta (SLO) | SLI (como medir) |
|----|----------|-----------|------------------|
| NFR-001 | Performance — cold start | p99 de boot→ready **≤ 250 ms** com pool quente; p99 **≤ 900 ms** sem pool quente. | `aios_runtime_cold_start_ms` (histogram) p99. |
| NFR-002 | Performance — overhead de sandbox | Overhead de CPU do sandbox **≤ 5%**; overhead de latência por passo **≤ 8 ms**. | `aios_runtime_sandbox_overhead_ratio`; comparação com baseline sem sandbox. |
| NFR-003 | Performance — passo do loop | p95 de latência de passo (excluindo tempo de LLM/tool) **≤ 15 ms**. | `aios_runtime_step_latency_ms` p95. |
| NFR-004 | Escalabilidade | **≥ 500** sessões concorrentes por nó (agentes ativos leves); **10⁶+** agentes majoritariamente *cold*, materializáveis sob demanda. | Contagem de sessões `busy`; taxa de resume. |
| NFR-005 | Disponibilidade | Runtime pool disponível **≥ 99,9%**; falha de instância NÃO perde trabalho aceito. | `aios_runtime_availability_ratio`. |
| NFR-006 | Recuperabilidade | **RTO ≤ 30 s** para retomar sessão suspensa; **RPO ≤ 1 passo** do loop (última observação persistida). | Tempo `resume` até estado ativo; perda máxima de passos. |
| NFR-007 | Throughput de eventos | **≥ 5.000 msg/s** de eventos de execução por nó sem backpressure. | `aios_runtime_events_published_total` rate. |
| NFR-008 | Segurança | 100% das ações privilegiadas passam pelo PEP; 0 escapes de sandbox em *pentest*. | Auditoria (025); testes de sandbox escape. |
| NFR-009 | Observabilidade | 100% dos passos com trace/span e correlação `traceparent`/`tenant`. | Cobertura de spans; amostragem OTel. |
| NFR-010 | Custo | Overhead de memória por sessão ociosa **≤ 30 MiB**; sessão *cold* consome **0** RAM. | `aios_runtime_session_rss_mib`. |
| NFR-011 | Manutenibilidade | Cobertura de testes **≥ 85%**; contratos gRPC/eventos testados por *contract tests*. | Relatório de cobertura; suíte de contrato. |
| NFR-012 | Isolamento | Nenhum vazamento cross-tenant; `tenant_id` validado contra contexto autenticado. | Testes de isolamento; RLS. |

---

## 8. Chaves de Configuração Principais

Escopo: **G**=global, **T**=por tenant, **A**=por agente. Recarregável em runtime
quando indicado. Variáveis de ambiente prefixadas `AIOS_RT_`.

| Chave | Default | Faixa | Escopo | Reload |
|-------|---------|-------|--------|--------|
| `runtime.pool.warm_size` | 32 | 0–1024 | G/T | sim |
| `runtime.pool.max_sessions_per_node` | 500 | 1–5000 | G | sim |
| `runtime.cold_start.target_ms` | 250 | 50–2000 | G | não |
| `loop.max_steps` | 50 | 1–1000 | T/A | sim |
| `loop.max_recursion_depth` | 8 | 1–64 | T/A | sim |
| `loop.max_tool_fanout` | 8 | 1–64 | T/A | sim |
| `loop.step_timeout_ms` | 30000 | 1000–600000 | T/A | sim |
| `loop.wall_clock_budget_ms` | 900000 | 1000–86400000 | T/A | sim |
| `budget.max_tokens` | 200000 | 100–10000000 | T/A | sim |
| `budget.max_cost_usd` | 5.00 | 0.01–1000 | T/A | sim |
| `model.default_timeout_ms` | 60000 | 1000–600000 | T | sim |
| `model.fallback_enabled` | true | bool | T | sim |
| `tool.invocation_timeout_ms` | 30000 | 1000–300000 | T/A | sim |
| `tool.circuit_breaker.error_threshold` | 5 | 1–100 | T | sim |
| `tool.retry.max_attempts` | 3 | 0–10 | T/A | sim |
| `tool.retry.backoff_base_ms` | 200 | 10–10000 | T | sim |
| `sandbox.profile` | `net-restricted` | `default\|net-restricted\|no-net\|custom` | T/A | não |
| `sandbox.cpu_millicores` | 1000 | 100–8000 | T/A | não |
| `sandbox.mem_mib` | 512 | 128–8192 | T/A | não |
| `sandbox.pids_max` | 256 | 16–4096 | T/A | não |
| `sandbox.fs_writable_mib` | 128 | 0–4096 | T/A | não |
| `sandbox.egress_allowlist` | `[]` | lista host:porta | T/A | sim |
| `checkpoint.enabled` | true | bool | T | sim |
| `checkpoint.interval_steps` | 5 | 1–100 | T/A | sim |
| `checkpoint.max_size_mib` | 64 | 1–1024 | T | sim |
| `events.outbox.flush_interval_ms` | 100 | 10–5000 | G | sim |
| `events.outbox.max_retries` | 10 | 0–100 | G | sim |
| `heartbeat.interval_ms` | 2000 | 500–30000 | G | sim |
| `watchdog.stuck_timeout_ms` | 120000 | 5000–600000 | G | sim |
| `telemetry.sampling_ratio` | 1.0 | 0.0–1.0 | G/T | sim |
| `pii.redaction.enabled` | true | bool | G/T | sim |
| `runtime.pep.decision_cache_ttl_ms` | 5000 | 0–60000 | T | sim |

---

## 9. Modos de Falha e Estratégia de Recuperação

FMEA resumido (detalhamento em `FailureRecovery.md`). Idempotência conforme
RFC-0001 §5.5; retries com backoff exponencial + *jitter*; DLQ em JetStream.

| Modo de falha | Detecção | Recuperação | Idempotência/Retry | RTO/RPO |
|---------------|----------|-------------|--------------------|---------|
| Falha de boot do sandbox | `RuntimeBootstrapper` erro | Marca instância `dead`; Supervisor recria; erro `AIOS-RUNTIME-0002`. | Boot idempotente por `Idempotency-Key`. | RTO ≤ 250 ms (novo slot quente). |
| Timeout/erro de tool (015) | Timeout/erro do ToolInvoker | Retry (até `max_attempts`); *circuit breaker*; fallback/replan. | Invocação idempotente por `idempotency_key`. | RPO 0 (passo re-executável). |
| Falha do Model Router (017) | Erro/timeout do ModelRouterClient | Fallback de modelo (se habilitado); retry; degradar contexto. | Chamada idempotente; dedup por `model_call_id`. | RPO 0. |
| Cota esgotada | QuotaGovernor | Suspende (checkpoint) ou aborta; `quota_exceeded`/`AIOS-RUNTIME-0004`. | — | Retomável via resume. |
| Loop travado/deadlock | Watchdog (`stuck_timeout_ms`) | Interrompe passo; tenta replan; se falhar, kill; `AIOS-RUNTIME-0008`. | — | RTO ≤ 30 s (resume de checkpoint). |
| Crash do processo runtime | Heartbeat perdido | Supervisor recria; sessão retomada do último checkpoint. | Outbox re-publica eventos pendentes (dedup por `event.id`). | RTO ≤ 30 s; RPO ≤ 1 passo. |
| Violação de sandbox | seccomp/monitor | Kill imediato; `sandbox_violation` (auditoria 025); `AIOS-RUNTIME-0015`. | — | Sessão isolada; sem contágio. |
| Perda de conexão NATS | Erro do EventPublisher | Bufferiza no outbox local; reconecta; re-publica. | Dedup por `event.id`. | RPO 0 (outbox durável efêmero). |
| Checkpoint corrompido | Checksum inválido | Rejeita resume; retoma do checkpoint anterior; `AIOS-RUNTIME-0010`. | Checksums versionados. | RPO ≤ intervalo de checkpoint. |
| Falha de 010/011 (memória/contexto) | Erro do cliente | Retry; degradar para contexto mínimo; se persistir, falhar sessão. | Leituras idempotentes. | RPO 0. |

Alinhamento global: RTO ≤ 15 min / RPO ≤ 5 min do plano de controle (V-R10) são
**limites superiores**; este módulo, por ser efêmero e checkpointado, mira metas
mais agressivas (NFR-006).

---

## 10. Escalabilidade e Concorrência

Estratégia central (herdada de `001-Architecture` §12): **agentes efêmeros e
*cold-storable*** + **sharding determinístico** + **comunicação seletiva**.

| Dimensão | Estratégia |
|----------|-----------|
| **Sharding** | `shard = hash(tenant_id, agent_id) mod N`; cada runtime pool atende um conjunto de shards. Sessões de um mesmo agente sempre no mesmo shard (afinidade), evitando checkpoints cross-shard. |
| **Pool quente** | `warm_size` runtimes pré-aquecidos por pool para cold start ≤ 250 ms; autoscaling guiado por fila do Scheduler + custo (026). |
| **Hibernação (*cold agent*)** | Agentes inativos são checkpointados para MinIO/PG e liberam RAM (NFR-010: 0 RAM em *cold*), materializados sob demanda. Base para 10⁶+ agentes. |
| **Concorrência intra-processo** | Loop cognitivo assíncrono (`asyncio`); I/O (LLM/tool/memória) não-bloqueante; um agente por processo (isolamento forte). Paralelismo de tools limitado por `max_tool_fanout`. |
| **Locks** | Sessão possui *lease* distribuído (Redis) por `session_id` para garantir **exatamente um** runtime ativo por sessão; `resume` só ocorre com aquisição do lease. Evita dupla execução. |
| **Backpressure** | Admissão limitada pelo Scheduler (009); no runtime, `max_sessions_per_node` e outbox com limites; JetStream aplica *flow control*. Ao saturar: rejeita com `AIOS-RUNTIME-0001`/`0013`. |
| **Fan-out de comunicação** | Eventos de execução agregados/amostrados (`telemetry.sampling_ratio`) para evitar O(N²); heartbeat com intervalo configurável. |
| **Isolamento de ruído** | *Bulkhead* por sandbox (`cgroups`); *noisy neighbor* contido por cotas de CPU/RAM/PIDs por agente e cotas de custo por tenant. |
| **Caminho de escala** | 1→10³: pool único; 10³→10⁵: múltiplos pools + sharding de sessão + NATS cluster; 10⁵→10⁶+: maioria *cold*, resume < 250 ms, pools distribuídos por AZ. |

---

## 11. ADRs a Propor e RFCs Relacionadas

Faixa reservada de ADR do módulo (007 × 10 = **ADR-0070 … ADR-0079**), sem colisão
com outros módulos.

| ADR | Título proposto |
|-----|-----------------|
| ADR-0070 | Runtime como processo Python por agente vs. threads/workers compartilhados (isolamento forte). |
| ADR-0071 | Mecanismo de sandbox: `seccomp-bpf` + `cgroups v2` + namespaces vs. gVisor/microVM (Firecracker). |
| ADR-0072 | Protocolo de controle Runtime↔Supervisor: gRPC (comando) + NATS (heartbeat/eventos). |
| ADR-0073 | Estratégia de cold start: pool quente pré-forkado + *copy-on-write* de intérprete. |
| ADR-0074 | Modelo de checkpoint/hibernação: serialização de working memory + cursor em MinIO. |
| ADR-0075 | MCP host embarcado no sandbox vs. sidecar externo. |
| ADR-0076 | Outbox transacional local (SQLite/WAL efêmero) para atomicidade estado↔evento. |
| ADR-0077 | Loop ReAct assíncrono (`asyncio`) e limites de passos/profundidade/fan-out. |
| ADR-0078 | PEP local com cache de *capabilities* e invalidação por evento de política. |
| ADR-0079 | *Lease* distribuído por sessão (Redis) para exatamente-um-ativo em resume. |

RFCs relacionadas: **RFC-0001** (baseline, aplicada integralmente); **RFC-0070**
(*Runtime↔Supervisor Control Protocol*, a propor); **RFC-0071** (*MCP Host
Sandboxing Profile*, a propor). Referências a `002-ADR/` (ADR-0001..0010 globais).

---

## 12. Decisões de Segurança

### 12.1 AuthN/AuthZ

- **AuthN entre serviços:** **mTLS** interno obrigatório (RFC-0001 §6, `021`). O
  runtime autentica o Supervisor e os serviços do control plane por certificado.
- **AuthZ:** O runtime é **PEP local** (`CapabilityEnforcer`). Toda ação
  privilegiada (tool, egress, syscall cognitiva) exige **capability token**
  concedido no boot (escopado por `tenant`/`agent`) e, quando a política exigir
  decisão dinâmica, consulta o **PDP** via Kernel/`022-Policy`. *Default deny*.
- **Isolamento de tenant:** `tenant_id` em URNs/subjects é fronteira; o runtime
  **NÃO DEVE** aceitar `tenant` divergente do contexto autenticado (RFC-0001 §6).
- **Segredos:** injetados pelo `021-Security` escopados por tenant, montados em
  tmpfs, nunca logados; rotação sem reinício quando possível.

### 12.2 Threat model STRIDE (resumido)

| Categoria | Ameaça | Mitigação |
|-----------|--------|-----------|
| **S**poofing | Runtime/Supervisor falso. | mTLS mútuo; validação de identidade de serviço; claims assinadas. |
| **T**ampering | Alteração de AgentSpec/checkpoint/eventos. | Checksums (sha256) em checkpoints; envelope assinado; outbox íntegro; FS read-only. |
| **R**epudiation | Ação de agente sem trilha. | Eventos imutáveis + auditoria (025); *provenance* por `ReActStep`; correlação `traceparent`. |
| **I**nformation disclosure | Vazamento de PII/cross-tenant em logs/eventos. | Redação de PII (`pii.redaction`); RLS por `tenant_id`; `detail` de erro sem dados sensíveis; egress *default deny*. |
| **D**enial of service | Agente consome recursos ilimitados (*noisy neighbor*). | `cgroups` (CPU/RAM/PIDs); cotas locais (tokens/USD/passos/wall); `max_sessions_per_node`; backpressure. |
| **E**levation of privilege | Escape de sandbox / syscall proibida / acesso a rede não permitido. | `seccomp-bpf` allowlist; namespaces; egress allowlist; kill em violação + `sandbox_violation`; sem `CAP_*` desnecessários. |

### 12.3 LGPD/GDPR

- **Minimização:** payloads de evento/log aplicam redação/tokenização de PII
  (RFC-0001 §7); `thought`/`observation` são redigidos conforme metadados de
  sensibilidade de `010-Memory`.
- **Base legal e retenção:** itens de memória com dados pessoais carregam metadados
  de base legal/retenção geridos por `010`/`025`; o runtime respeita esses metadados
  ao persistir working memory em checkpoint.
- **Direito ao esquecimento:** operação de expurgo rastreável — o runtime invalida
  checkpoints/working memory sob comando de expurgo e emite evento auditável.
- **Localidade:** perfis de sandbox podem restringir egress a regiões permitidas
  por tenant (residência de dados).

---

## Rastreabilidade e Reserva de Namespaces (para os próximos agentes)

| Recurso | Reserva deste módulo |
|---------|----------------------|
| Domínio de erro | `AIOS-RUNTIME-<NNNN>`, faixa **0001–0099** (definidos 0001–0015). |
| Faixa de ADR | **ADR-0070 … ADR-0079**. |
| RFCs propostas | **RFC-0070**, **RFC-0071**. |
| Pacotes gRPC | `aios.runtime.v1` (`RuntimeControl`, `AgentExecution`). |
| Subjects emitidos | `aios.<t>.agent.runtime.*`, `aios.<t>.task.execution.*`. |
| Tipo URN proposto | `runtime` (via ADR-0070; consome `agent\|task\|tool\|model\|plan`). |

*Fim do Design Brief do Módulo 007 · Agent Runtime. Este documento é a base
normativa para os 26 documentos obrigatórios em `../_templates/MODULE_TEMPLATE.md`.*
