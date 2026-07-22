---
Documento: FunctionalRequirements
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0070, ADR-0071, ADR-0072, ADR-0073, ADR-0074, ADR-0075, ADR-0076, ADR-0077, ADR-0078, ADR-0079
RFCs relacionados: RFC-0001, RFC-0070, RFC-0071
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `Requirements.md`, `../003-RFC/RFC-0001-Architecture-Baseline.md`
---

# 007-Agent-Runtime — Functional Requirements

> Convenção de prioridade: **MoSCoW** (`Must` / `Should` / `Could` / `Won't-this-release`).
> Todo FR é mensurável e verificável por um critério de aceite objetivo. A
> numeração (`FR-001`..`FR-018`) é a **mesma** fixada em `_DESIGN_BRIEF.md`
> §7.1 e em `Requirements.md` §3 — não é redefinida aqui, apenas elaborada com
> critério de aceite detalhado. Origem aponta para a seção do brief de onde o
> requisito deriva.

## A. Materialização e Ciclo de Vida do Agente

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-001 | O `RuntimeBootstrapper` DEVE materializar um agente a partir de um **AgentSpec** recebido via `Boot` (`aios.runtime.v1.RuntimeControl`), preparando sandbox, validando *capabilities* e conduzindo a `AgentSession` até um estado terminal (`done`/`failed`) ou quiescente (`suspended`). | Must | Chamada `Boot` bem-sucedida cria `RuntimeInstance`/`AgentSession`, emite `aios.<t>.agent.runtime.booted` e transiciona `boot→ready`; AgentSpec inválido retorna `AIOS-RUNTIME-0009` sem criar sessão. | Brief §1.2 (R-01), §5.1 |
| FR-009 | O `CheckpointManager` DEVE gerar checkpoints íntegros (checksum sha256) do estado do agente (working memory + `step_cursor` + plano) permitindo suspensão e retomada (*cold agent*) sem perda de continuidade cognitiva. | Must | `Suspend` gera `Checkpoint` com `checksum` válido e transiciona a sessão a `suspended`; `Resume` subsequente restaura `working_memory_ref`, `plan_snapshot_ref` e `step_cursor` idênticos aos do momento da suspensão (teste de round-trip). | Brief §1.2 (R-09), §3.2 (Checkpoint), §4.1 |
| FR-018 | O `SupervisorChannel`/`HealthProbe` DEVEM reportar heartbeat periódico (`heartbeat.interval_ms`) e expor `readyz`/`healthz` ao Runtime Supervisor, incluindo estado (`RuntimeState`), consumo (`consumed`) e orçamento (`budget`) correntes. | Must | Evento `aios.<t>.agent.runtime.heartbeat` publicado a cada `heartbeat.interval_ms` com `runtime_id`, `state`, `consumed`, `budget`; `GET /v1/runtime/healthz` e `/readyz` respondem conforme o estado real do `RuntimeState` (§4.3 do brief). | Brief §1.2 (R-11), §5.3, §4.3 |

## B. Loop Cognitivo ReAct

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-002 | O `CognitiveLoopEngine` DEVE sequenciar o loop ReAct `perceive→think→act→observe→repetir`, produzindo a cada iteração um `ReActStep` imutável com `phase`, `status` e `latency_ms`. | Must | Cada iteração completa do loop gera exatamente um `ReActStep` persistido/publicado; evento `aios.<t>.task.execution.step` emitido com `step_id`, `phase`, `status`, `latency_ms`; transições seguem a FSM de `StateMachine.md` §1. | Brief §1.2 (R-02), §3.2 (ReActStep), §4.1 |
| FR-006 | O `CognitiveLoopEngine` DEVE detectar falha recuperável ou deriva de objetivo e delegar replanejamento ao `PlanningClient` (`012-Planning`), transicionando `thinking→reflecting→thinking` com um `plan_urn` atualizado. | Must | Falha marcada `recuperável` (ex.: tool timeout com retries esgotados, mas orçamento disponível) aciona `reflecting`; plano recebido de `012` atualiza `AgentSession.plan_urn` e o loop retoma em `thinking`; sem plano viável, transição para `failed` (`AIOS-RUNTIME-0014`). | Brief §1.2 (R-02), §4.2 (T6, T10, T11) |
| FR-016 | O `HealthProbe` DEVE operar como *watchdog* do loop, detectando passos travados (*stuck*/deadlock) via `watchdog.stuck_timeout_ms` e acionando replanejamento ou `kill`. | Should | Passo sem progresso além de `watchdog.stuck_timeout_ms` emite `AIOS-RUNTIME-0008`; o watchdog tenta replanejamento antes de encerrar a sessão (`terminating`); teste de *chaos* simula um passo travado e confirma a recuperação/encerramento dentro do RTO de NFR-006. | Brief §1.2 (R-02), §9 |
| FR-017 | O `QuotaGovernor` DEVE limitar `loop.max_steps`, `loop.max_recursion_depth` e `loop.max_tool_fanout`, rejeitando (e não apenas registrando) a continuação do loop ao exceder qualquer limite. | Must | Excedente de qualquer limite retorna `AIOS-RUNTIME-0012` e força transição a `failed` (ou `suspending`, conforme política); teste de integração confirma rejeição determinística no limite exato configurado. | Brief §1.2 (R-10), §8 (`loop.max_steps`, `loop.max_recursion_depth`, `loop.max_tool_fanout`) |

## C. Integração com Model Router e Tool Manager

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-003 | O `ReasoningModule` DEVE realizar **toda** inferência de LLM através do `ModelRouterClient → 017-Model-Router`; nenhum componente do runtime DEVE se conectar diretamente a um provedor de modelo. | Must | Contract test estático/dinâmico confirma zero chamadas de rede a domínios de provedor de LLM fora do cliente `ModelRouterClient`; toda linha de `ModelCall` tem `resolved_model_urn` preenchido pelo Router. | Brief §1.2 (R-04), §2.2 (regra de fronteira) |
| FR-004 | O `ActionExecutor`/`ToolInvoker` DEVEM realizar **toda** invocação de ferramenta através de `015-Tool-Manager`/MCP host; nenhum componente DEVE executar uma ferramenta externa por conta própria. | Must | Contract test confirma zero invocações de ferramenta fora do `ToolInvoker`; toda linha de `ToolInvocation` tem `tool_urn` e `capability_id` validados antes da execução. | Brief §1.2 (R-03), §2.2 (regra de fronteira) |
| FR-014 | O `ToolInvoker` e o `ModelRouterClient` DEVEM aplicar *circuit breaker*, *retry* com backoff exponencial + *jitter*, e timeout configurável em toda chamada a `015`/`017`. | Should | Falha transitória é reintentada até `tool.retry.max_attempts`/config equivalente de modelo; abertura de *circuit breaker* é reportada como `result_status=circuit_open`/`status=fallback`; teste de *chaos* confirma fechamento do breaker após recuperação (NFR-016). | Brief §1.2 (R-13), §8 (`tool.circuit_breaker.*`, `tool.retry.*`, `model.fallback_enabled`) |

## D. Memória e Contexto

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-005 | O `PerceptionModule` DEVE montar o *percept* de cada passo recuperando itens de memória via `MemoryClient → 010-Memory` e contexto via `ContextClient → 011-Context`, sem acesso direto a PostgreSQL/pgvector/AGE. | Must | Percept do passo contém `contexto` proveniente de `011-Context` e `memória` proveniente de `010-Memory`; teste estático confirma ausência de driver de banco de dados no processo runtime. | Brief §1.2 (R-05), §2.2 (regra de fronteira) |

## E. Isolamento, Sandbox e Segurança

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-007 | O `SandboxManager` DEVE isolar cada agente materializado com namespaces de processo, `seccomp-bpf`, `cgroups v2` (CPU/RAM/PIDs), filesystem read-only + escrita efêmera, e *default deny* de egress de rede salvo allowlist explícita. | Must | Syscall fora do perfil `seccomp` gera `AIOS-RUNTIME-0015` + evento `sandbox_violation` + `kill` imediato; teste de *pentest*/integração confirma ausência de escape em 100% dos cenários testados (NFR-008). | Brief §1.2 (R-06), §3.2 (SandboxProfile), §12.2 |
| FR-008 | O `CapabilityEnforcer` DEVE atuar como PEP local para toda ação privilegiada (tool, egress, syscall cognitiva), validando a *capability* concedida no boot e, quando exigido, consultando o PDP via Kernel/`022-Policy`, com postura *default deny*. | Must | Ação sem capability válida retorna `AIOS-RUNTIME-0003` (HTTP 403) e não produz efeito colateral; auditoria de cobertura confirma que 100% das ações privilegiadas passam pelo `CapabilityEnforcer` (NFR-008). | Brief §1.2 (R-07), §12.1 |
| FR-015 | O runtime DEVE aplicar redação de PII em `thought`, `observation` e demais payloads de evento/log, conforme metadados de sensibilidade de `010-Memory`, antes de qualquer emissão externa. | Must | Auditoria de amostragem confirma ausência de PII em payloads publicados quando `pii.redaction.enabled=true`; teste de integração injeta PII sintética e verifica redação em `ReActStep.thought`/`.observation` antes da publicação. | Brief §1.2 (R-14), §12.3 |

## F. Cotas e Governança de Recursos Locais

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-010 | O `QuotaGovernor` DEVE honrar cotas locais de tokens, custo USD, wall-clock e passos do loop, e DEVE suspender (com checkpoint) ou abortar a sessão ao esgotar qualquer cota. | Must | Esgotamento de qualquer cota emite `AIOS-RUNTIME-0004` e `aios.<t>.agent.runtime.quota_exceeded`; sessão transiciona a `suspending`→`suspended` (se checkpoint possível) ou `terminating`→`failed`, nunca continuando a consumir orçamento além do limite. | Brief §1.2 (R-10), §8 (`budget.*`, `loop.wall_clock_budget_ms`) |

## G. Eventos, Idempotência e Streaming

| ID | Descrição | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-011 | O `EventPublisher`/`TelemetryEmitter` DEVEM emitir todos os eventos NATS do catálogo (`Events.md` §1) e telemetria OTel correlacionada (`traceparent`/`tenant`/`agent`), com atomicidade estado↔evento via **outbox**. | Must | 100% das transições de `StateMachine.md` publicam o evento correspondente; inspeção de spans confirma correlação completa; teste de *crash injection* no outbox não perde eventos pendentes (RPO ≤ 1 passo, NFR-006/NFR-009). | Brief §1.2 (R-08), §6 |
| FR-012 | `SubmitTask` DEVE ser idempotente por `Idempotency-Key`: repetições da mesma submissão dentro da janela de retenção DEVEM retornar o mesmo resultado sem novo efeito colateral. | Must | Repetir `SubmitTask` com a mesma `Idempotency-Key` e mesmo payload retorna o mesmo `session_id`/stream de resultado, sem nova `AgentSession` nem novo consumo de cota; chave reused com payload divergente retorna `AIOS-RUNTIME-0007`. | Brief §1.2 (R-12), §5.1, §5.2, RFC-0001 §5.5 |
| FR-013 | `SubmitTask`/`StreamSteps` DEVEM entregar o resultado e os passos ReAct como *stream* ordenado ao control plane. | Should | `SubmitTask` retorna `stream ExecutionEvent` em ordem de `step.index`; `StreamSteps` reflete o mesmo fluxo para observadores externos (ex.: WebConsole via control plane) sem reordenar ou omitir passos. | Brief §1.2 (R-11), §5.2 |

## Notas de rastreabilidade

- A relação completa `FR-NNN → UC-NNN → Teste` está em `Requirements.md` §5.
- Nenhum FR deste documento contradiz o `_DESIGN_BRIEF.md` §7.1; a numeração,
  o texto normativo e a prioridade MoSCoW são idênticos ao brief — este
  documento apenas acrescenta critério de aceite objetivo e origem detalhada.
- Alterações a este catálogo (adição, depreciação) DEVEM ser acompanhadas de
  atualização da matriz de rastreabilidade em `Requirements.md` e, quando
  aplicável, do `_DESIGN_BRIEF.md` primeiro (fonte única de verdade).
