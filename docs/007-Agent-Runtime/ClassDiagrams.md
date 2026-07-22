---
Documento: ClassDiagrams
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0074, ADR-0075, ADR-0076, ADR-0077, ADR-0078, ADR-0079
RFCs relacionados: RFC-0001, RFC-0070, RFC-0071
Depende de: ./Architecture.md, ./StateMachine.md, ../003-RFC/RFC-0001-Architecture-Baseline.md
---

# 007-Agent-Runtime — Diagramas de Classe / Estrutura

> Este documento detalha as estruturas de dados e interfaces internas do
> Agent Runtime em notação ASCII (adaptação de UML). Os campos e tipos
> reproduzem exatamente o modelo de dados de `./_DESIGN_BRIEF.md` §3 —
> nenhum campo é inventado ou renomeado. As interfaces de componente
> refletem `./Architecture.md` §4–§5. Tipos usam sintaxe Python (o runtime é
> implementado em Python 3.12+/`asyncio`, ADR-0070) com anotações de tipo
> (`Optional[T]`, `list[T]`, `dict[K, V]`).

## Índice

1. Convenções de notação
2. Entidades de domínio (estruturas de dados)
3. Interfaces públicas dos componentes
4. Contratos (pré-condições, pós-condições)
5. Invariantes
6. Diagrama de relações (composição/agregação/dependência)
7. Enumerações e Value Objects
8. Referências

---

## 1. Convenções de Notação

```
┌─────────────────────────┐
│      NomeDaClasse        │   « stereotype »  ex.: «entity», «interface», «value object»
├─────────────────────────┤
│ + campoPublico : Tipo    │   + público   - privado   # protegido
│ - campoPrivado : Tipo    │
├─────────────────────────┤
│ + Metodo(param: Tipo)    │
│     : TipoRetorno        │
└─────────────────────────┘

  A ──▶ B      dependência (A usa B)
  A ──◇ B      agregação (A contém referência a B, ciclo de vida independente)
  A ──◆ B      composição (A possui B, ciclo de vida acoplado)
  A ──▷ B      A implementa a interface B
  A ──1..* B   cardinalidade
```

---

## 2. Entidades de Domínio (Estruturas de Dados)

### 2.1 `AgentSession` («entity», efêmera no runtime, durável no control plane)

```
┌──────────────────────────────────────────────────────────────────┐
│                     AgentSession  «entity»                        │
├──────────────────────────────────────────────────────────────────┤
│ + session_id       : Ulid                                         │
│ + agent_urn        : Urn        // urn:aios:<tenant>:agent:<ULID> │
│ + task_urn         : Urn                                          │
│ + tenant_id         : str        // RLS boundary                  │
│ + runtime_id        : Ulid       // FK → RuntimeInstance           │
│ + state             : AgentExecState                              │
│ + plan_urn          : Optional[Urn]                                │
│ + step_cursor       : int        // default 0                     │
│ + budget            : ResourceBudget                               │
│ + consumed          : ResourceUsage                                │
│ + idempotency_key   : str        // unique(tenant, task)          │
│ + traceparent       : str        // W3C Trace Context             │
│ + created_at        : datetime                                    │
│ + updated_at        : datetime                                    │
│ + terminal_reason   : Optional[str]                                 │
├──────────────────────────────────────────────────────────────────┤
│ + can_transition_to(target: AgentExecState) -> bool                │
│ + is_over_budget() -> bool     // consumed > budget, por dimensão  │
└──────────────────────────────────────────────────────────────────┘
```

Invariantes locais: ver §5 (I1–I4). Fonte da verdade **durável** reside no
control plane (`010-Memory`/PostgreSQL, RLS); o runtime mantém a instância
ativa em memória de processo mais o cursor no outbox local.

### 2.2 `ResourceBudget` / `ResourceUsage` («value object», subcampos jsonb de `budget`/`consumed`)

```
┌───────────────────────────────────────┐  ┌───────────────────────────────────────┐
│   ResourceBudget  «value object»       │  │   ResourceUsage  «value object»        │
├───────────────────────────────────────┤  ├───────────────────────────────────────┤
│ + max_tokens        : int              │  │ + tokens_used       : int              │
│ + max_cost_usd       : Decimal          │  │ + cost_usd_spent     : Decimal          │
│ + wall_clock_budget_ms : int            │  │ + wall_clock_elapsed_ms : int           │
│ + max_steps          : int              │  │ + steps_executed     : int              │
│ + max_recursion_depth : int             │  │ + current_depth      : int              │
│ + max_tool_fanout     : int             │  │ + concurrent_tool_calls : int           │
└───────────────────────────────────────┘  └───────────────────────────────────────┘
```

### 2.3 `ReActStep` («entity», event-sourced, append-only)

```
┌──────────────────────────────────────────────────────────────────┐
│                       ReActStep  «entity»                         │
├──────────────────────────────────────────────────────────────────┤
│ + step_id            : Ulid      // ordenável por tempo           │
│ + session_id          : Ulid      // FK → AgentSession             │
│ + tenant_id           : str                                        │
│ + index               : int       // ordem no loop                 │
│ + phase               : ReActPhase                                 │
│ + percept_ref         : Optional[str]                               │
│ + thought             : Optional[str]   // redigido para PII       │
│ + action_type         : Optional[ActionType]                        │
│ + tool_invocation_id  : Optional[Ulid]                               │
│ + model_call_id       : Optional[Ulid]                               │
│ + observation         : Optional[dict]                              │
│ + status              : StepStatus                                  │
│ + latency_ms          : int                                        │
│ + created_at          : datetime                                   │
├──────────────────────────────────────────────────────────────────┤
│ + is_terminal_decision() -> bool  // action_type in {final_answer}  │
└──────────────────────────────────────────────────────────────────┘
```

### 2.4 `ToolInvocation` / `ModelCall` («entity»)

```
┌────────────────────────────────────────┐  ┌────────────────────────────────────────┐
│    ToolInvocation  «entity»              │  │    ModelCall  «entity»                   │
├────────────────────────────────────────┤  ├────────────────────────────────────────┤
│ + tool_invocation_id : Ulid              │  │ + model_call_id     : Ulid               │
│ + session_id          : Ulid              │  │ + session_id         : Ulid               │
│ + tenant_id           : str                │  │ + tenant_id          : str                 │
│ + tool_urn            : Urn                │  │ + requested_model    : Optional[str]        │
│ + capability_id        : str                │  │ + resolved_model_urn : Urn                  │
│ + arguments_ref        : str                │  │ + prompt_tokens      : int                  │
│ + result_status        : ToolResultStatus    │  │ + completion_tokens  : int                  │
│ + retries              : int                │  │ + cost_usd           : Decimal               │
│ + cost_usd             : Decimal             │  │ + status             : ModelCallStatus       │
│ + latency_ms           : int                │  │ + latency_ms         : int                  │
│ + idempotency_key      : str                │  │ + created_at         : datetime              │
│ + created_at           : datetime            │  └────────────────────────────────────────┘
└────────────────────────────────────────┘
```

### 2.5 `Checkpoint` / `SandboxProfile` («entity»)

```
┌────────────────────────────────────────┐  ┌────────────────────────────────────────┐
│     Checkpoint  «entity»                  │  │   SandboxProfile  «entity»                │
├────────────────────────────────────────┤  ├────────────────────────────────────────┤
│ + checkpoint_id      : Ulid               │  │ + sandbox_profile_id : str                │
│ + session_id          : Ulid               │  │ + tenant_id           : Optional[str]       │
│ + tenant_id           : str                 │  │ + seccomp_profile     : dict                │
│ + step_cursor         : int                 │  │ + cpu_millicores      : int                 │
│ + working_memory_ref   : str  // MinIO      │  │ + mem_mib             : int                 │
│ + plan_snapshot_ref    : Optional[str]        │  │ + pids_max            : int                 │
│ + size_bytes           : int                 │  │ + egress_allowlist    : list[str]            │
│ + checksum             : str  // sha256      │  │ + fs_writable_mib     : int                 │
│ + created_at           : datetime            │  └────────────────────────────────────────┘
└────────────────────────────────────────┘
```

Invariante: `Checkpoint.checksum` DEVE ser recalculado e validado antes de
qualquer `Resume`; falha de validação impede a restauração (ver §5, I5).

---

## 3. Interfaces Públicas dos Componentes

> Cada interface corresponde a um componente de `./Architecture.md` §5.
> Assinaturas são `async` por padrão (loop cognitivo não-bloqueante,
> ADR-0077).

```
┌──────────────────────────────────────────────────────────┐
│         IExecutionStateMachine  «interface»                │
├──────────────────────────────────────────────────────────┤
│ + current_state() -> AgentExecState                        │
│ + transition(event: TransitionEvent) -> AgentExecState     │
│     // valida guarda, aplica ação de saída/entrada,        │
│     // publica evento via EventPublisher                   │
│ + is_terminal() -> bool                                    │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│         ICognitiveLoopEngine  «interface»                  │
├──────────────────────────────────────────────────────────┤
│ + async run_step(session: AgentSession) -> ReActStep       │
│ + async run_until_terminal(session: AgentSession)          │
│     -> AsyncIterator[ReActStep]                            │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│         IQuotaGovernor  «interface»                         │
├──────────────────────────────────────────────────────────┤
│ + check_budget(session: AgentSession,                       │
│     dimension: QuotaDimension) -> QuotaDecision             │
│ + record_consumption(session_id: Ulid,                      │
│     usage_delta: ResourceUsage) -> None                     │
│ + is_exhausted(session: AgentSession) -> bool                │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│         ICapabilityEnforcer  «interface» (PEP local)        │
├──────────────────────────────────────────────────────────┤
│ + async authorize(subject: AgentIdentity,                   │
│     action: str, resource: str) -> Decision                 │
│     // default deny; consulta PDP em cache-miss             │
│ + invalidate_cache(session_id: Optional[Ulid]) -> None       │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│         IToolInvoker  «interface»                            │
├──────────────────────────────────────────────────────────┤
│ + async invoke(session: AgentSession, tool_urn: Urn,          │
│     arguments: dict, idempotency_key: str)                   │
│     -> ToolInvocationResult                                  │
│     // circuit breaker + retry + timeout internos            │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│         IModelRouterClient  «interface»                      │
├──────────────────────────────────────────────────────────┤
│ + async infer(session: AgentSession, prompt: PromptBundle)    │
│     -> ModelCallResult                                       │
│     // streaming opcional; fallback interno                  │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│         ICheckpointManager  «interface»                      │
├──────────────────────────────────────────────────────────┤
│ + async create(session: AgentSession) -> Checkpoint            │
│ + async restore(checkpoint_id: Ulid) -> RestoredState          │
│     // valida checksum antes de retornar (I5)                │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│         IEventPublisher  «interface»                         │
├──────────────────────────────────────────────────────────┤
│ + async publish(event_type: str, subject: Urn,                │
│     data: dict, tenant: str) -> None                          │
│     // grava em outbox local antes de publicar (ADR-0076)    │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│         ISupervisorChannel  «interface»                      │
├──────────────────────────────────────────────────────────┤
│ + async boot(spec: AgentSpec) -> AgentSession                 │
│ + async suspend(session_id: Ulid) -> Checkpoint                │
│ + async resume(checkpoint_id: Ulid) -> AgentSession             │
│ + async kill(session_id: Ulid) -> None                          │
│ + async drain() -> None                                         │
│ + async get_status() -> RuntimeStatus                           │
└──────────────────────────────────────────────────────────┘
```

---

## 4. Contratos (Pré-condições, Pós-condições)

| Componente/Método | Pré-condição | Pós-condição |
|--------------------|---------------|----------------|
| `IExecutionStateMachine.transition` | Evento e guarda válidos para o estado corrente (ver `./StateMachine.md` §2). | Estado atualizado; evento correspondente publicado; nenhuma transição parcial visível a chamadores concorrentes. |
| `ICognitiveLoopEngine.run_step` | `AgentSession.state` permite um passo (`ready`/`thinking`); `QuotaGovernor.check_budget()` não retorna `exhausted`. | Exatamente um `ReActStep` novo produzido; `step_cursor` incrementado em 1. |
| `IQuotaGovernor.check_budget` | Nenhuma. | Retorna `QuotaDecision` determinística (não modifica estado); efeitos colaterais (suspensão/abort) são decididos pelo chamador. |
| `ICapabilityEnforcer.authorize` | `subject` autenticado com `tenant_id` do contexto. | Retorna `Decision` (`allow`/`deny`); em `deny`, nenhuma ação a jusante é executada pelo chamador. |
| `IToolInvoker.invoke` | `authorize()` já retornou `allow` para a tool-alvo. | `ToolInvocationResult` com `result_status` determinado; `ToolInvocation` registrado; `consumed.cost_usd` atualizado. |
| `IModelRouterClient.infer` | `session` em `thinking`. | `ModelCall` registrado com `status` determinado; nunca contata um provedor diretamente (FR-003). |
| `ICheckpointManager.create` | `checkpoint.enabled = true`. | `Checkpoint` com `checksum` válido persistido antes de a sessão transicionar a `suspended`. |
| `ICheckpointManager.restore` | `Checkpoint.checksum` recalculado bate com o armazenado. | `RestoredState` com `working_memory`/`plan`/`step_cursor` consistentes; falha de checksum impede restauração (retorna erro, não estado parcial). |
| `IEventPublisher.publish` | Nenhuma. | Evento gravado em outbox local **antes** de qualquer publicação de rede (atomicidade estado↔evento, ADR-0076). |

---

## 5. Invariantes

| # | Invariante | Onde é imposta |
|---|------------|------------------|
| I1 | `AgentSession.state` DEVE sempre corresponder a um estado válido de `AgentExecState` (`./StateMachine.md`); nunca um valor fora do enum. | `ExecutionStateMachine` |
| I2 | `consumed ≤ budget` DEVE ser mantido a cada passo; violação força transição a `suspending`/`terminating`. | `QuotaGovernor` |
| I3 | Toda ação privilegiada (`tool_call`, egress, syscall cognitiva) DEVE passar por `CapabilityEnforcer.authorize()` antes de qualquer efeito colateral. | `CapabilityEnforcer`, `ActionExecutor`, `ToolInvoker` |
| I4 | `ReActStep.index` DEVE ser estritamente crescente e contíguo dentro de uma `session_id` (sem lacunas nem duplicatas). | `ObservationCollector` |
| I5 | `Checkpoint.checksum` (sha256) DEVE ser válido antes de qualquer `restore()`; checksum inválido impede a restauração e preserva o checkpoint íntegro anterior. | `CheckpointManager` |
| I6 | Um `session_id` DEVE ter, no máximo, **um** *lease* Redis ativo simultâneo — exatamente uma instância de runtime executando aquela sessão. | *Lease* distribuído (ADR-0079), verificado no `resume` |
| I7 | Estados terminais (`done`, `failed`) NÃO transicionam para nenhum outro estado. | `ExecutionStateMachine` |
| I8 | Todo evento publicado DEVE ter passado pelo outbox local antes de qualquer tentativa de publicação de rede. | `EventPublisher` |

---

## 6. Diagrama de Relações (Composição / Agregação / Dependência)

```
RuntimeBootstrapper ──◆ SandboxManager
RuntimeBootstrapper ──▶ CapabilityEnforcer
RuntimeBootstrapper ──▶ SupervisorChannel

ExecutionStateMachine ──▶ EventPublisher
ExecutionStateMachine ──▶ TelemetryEmitter
ExecutionStateMachine ──1..1 AgentSession

CognitiveLoopEngine ──◆ PerceptionModule
CognitiveLoopEngine ──◆ ReasoningModule
CognitiveLoopEngine ──◆ ActionExecutor
CognitiveLoopEngine ──◆ ObservationCollector
CognitiveLoopEngine ──▶ PlanningClient
CognitiveLoopEngine ──▶ QuotaGovernor
CognitiveLoopEngine ──1..* ReActStep

PerceptionModule ──▶ ContextClient
PerceptionModule ──▶ MemoryClient

ReasoningModule ──▶ ModelRouterClient
ReasoningModule ──▶ ContextClient
ReasoningModule ──1..* ModelCall

ActionExecutor ──◇ McpHost
ActionExecutor ──▶ ToolInvoker
ActionExecutor ──▶ CapabilityEnforcer
ToolInvoker ──1..* ToolInvocation

ObservationCollector ──▶ MemoryClient
ObservationCollector ──▶ CheckpointManager

McpHost ──◆ ToolInvoker
McpHost ──▶ SandboxManager

CheckpointManager ──1..* Checkpoint
CheckpointManager ──▶ MemoryClient

QuotaGovernor ──1..1 ResourceBudget
QuotaGovernor ──1..1 ResourceUsage

CapabilityEnforcer ──▷ IPolicyDecisionCache
SandboxManager ──1..1 SandboxProfile

EventPublisher ──▶ OutboxStore  (SQLite/WAL local, ADR-0076)
SupervisorChannel ──▶ HealthProbe
HealthProbe ──▶ ExecutionStateMachine   (watchdog observa estado)
```

---

## 7. Enumerações e Value Objects

```
enum AgentExecState:
    BOOT, READY, THINKING, ACTING, WAITING_IO, REFLECTING,
    DONE, FAILED, SUSPENDING, SUSPENDED, RESUMING, TERMINATING

enum RuntimeState:
    WARMING, IDLE, BUSY, DRAINING, DEAD

enum ReActPhase:
    PERCEIVE, THINK, ACT, OBSERVE, REFLECT

enum ActionType:
    TOOL_CALL, FINAL_ANSWER, REPLAN, NOOP

enum StepStatus:
    OK, ERROR, TIMEOUT, DENIED

enum ToolResultStatus:
    OK, ERROR, TIMEOUT, DENIED, CIRCUIT_OPEN

enum ModelCallStatus:
    OK, ERROR, TIMEOUT, FALLBACK

enum QuotaDimension:
    TOKENS, COST_USD, WALL_CLOCK_MS, STEPS, RECURSION_DEPTH, TOOL_FANOUT

value object Decision:
    outcome: Literal["allow", "deny"]
    reason: Optional[str]
    source: Literal["cache", "pdp"]

value object QuotaDecision:
    allowed: bool
    dimension: Optional[QuotaDimension]
    remaining: Optional[Decimal]
```

---

## 8. Referências

- Modelo de dados canônico (fonte): `./_DESIGN_BRIEF.md` §3.
- Máquina de estados que impõe I1/I2/I7: `./StateMachine.md`.
- Componentes e responsabilidades: `./Architecture.md` §4–§5.
- Contratos formais gRPC (`RuntimeControl`/`AgentExecution`): `./API.md`.
- Convenções de identidade (URN) e idempotência: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.1, §5.5.
