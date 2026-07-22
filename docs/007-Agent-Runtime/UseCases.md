---
Documento: UseCases
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0070, ADR-0071, ADR-0072, ADR-0073, ADR-0074, ADR-0076, ADR-0077, ADR-0078, ADR-0079
RFCs relacionados: RFC-0001, RFC-0070, RFC-0071
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `FunctionalRequirements.md`, `NonFunctionalRequirements.md`, `../008-Agent-Lifecycle/`, `../009-Scheduler/`, `../015-Tool-Manager/`, `../017-Model-Router/`, `../022-Policy/`
---

# 007-Agent-Runtime — Use Cases

> Convenção: cada caso de uso é numerado `UC-NNN`, descreve **ator**,
> **pré-condições**, **fluxo principal**, **fluxos alternativos**,
> **exceções** e **pós-condições**. Códigos de erro (`AIOS-RUNTIME-<NNNN>`) e
> eventos seguem o catálogo do `_DESIGN_BRIEF.md` §5.4/§6 e o envelope RFC
> 7807/CloudEvents de `../003-RFC/RFC-0001-Architecture-Baseline.md`. A
> rastreabilidade `UC → FR/NFR → Teste` está em `Requirements.md` §5.

## Índice

| UC | Título | FR/NFR primários | Método(s) `aios.runtime.v1` |
|----|--------|-------------------|-------------------------------|
| UC-001 | Boot de agente com pool quente (fluxo feliz) | FR-001, FR-018, NFR-001 | `Boot`, `GetStatus` |
| UC-002 | Boot de agente sem pool quente (cold boot) | FR-001, NFR-001 | `Boot` |
| UC-003 | Execução completa de um passo ReAct | FR-002, FR-003, FR-004, FR-005, FR-015 | `SubmitTask` (stream) |
| UC-004 | Loop com múltiplos passos até resposta final | FR-002 | `SubmitTask` (stream) |
| UC-005 | Replanejamento por falha recuperável | FR-006 | `SubmitTask` (stream) |
| UC-006 | Limite de passos/profundidade/fan-out excedido | FR-017 | `SubmitTask` (stream) |
| UC-007 | Esgotamento de cota local com suspensão e checkpoint | FR-009, FR-010 | `Suspend` |
| UC-008 | Crash do processo entre checkpoints e retomada | FR-009 | `Resume` |
| UC-009 | Perda de heartbeat e recriação pelo Supervisor | FR-018 | (interno; heartbeat) |
| UC-010 | Violação de sandbox (tentativa de escape) | FR-007 | (interno; watchdog seccomp) |
| UC-011 | Negação de capability pelo PEP local | FR-008 | `SubmitTask` (stream) |
| UC-012 | Indisponibilidade do Model Router e fallback | FR-003, FR-014 | `SubmitTask` (stream) |
| UC-013 | Indisponibilidade do Tool Manager e circuit breaker | FR-004, FR-014 | `SubmitTask` (stream) |
| UC-014 | Repetição idempotente de `SubmitTask` | FR-012 | `SubmitTask` |
| UC-015 | Loop travado detectado pelo watchdog | FR-016 | (interno; watchdog) |
| UC-016 | Streaming de passos ao control plane | FR-013 | `StreamSteps` |
| UC-017 | Isolamento de tenant (rejeição de tenant divergente) | NFR-012 | qualquer método `aios.runtime.v1` |
| UC-018 | Compatibilidade de ABI entre versões major | NFR-014 | qualquer método `aios.runtime.v1` |

---

## UC-001 — Boot de agente com pool quente (fluxo feliz)

**Ator primário:** Runtime Supervisor (`.NET 10`, `008-Agent-Lifecycle`).
**Atores secundários:** `SandboxManager`, `CapabilityEnforcer`, `McpHost`.

**Pré-condições:**
- P1. Existe ao menos uma instância `warming`/`idle` no pool quente (`runtime.pool.warm_size > 0`).
- P2. O `AgentSpec` é válido e contém *capabilities* concedidas para o agente.
- P3. O chamador fornece `Idempotency-Key` e cabeçalhos de correlação (RFC-0001 §5.6).

**Fluxo principal:**
1. O Supervisor invoca `Boot(BootRequest)` em uma instância `idle` do pool quente.
2. O `RuntimeBootstrapper` carrega o `AgentSpec`, cria `AgentSession` (`session_id`) e associa a `RuntimeInstance` (`runtime_id`).
3. O `SandboxManager` aplica o `SandboxProfile` (namespaces, `seccomp-bpf`, `cgroups v2`, FS efêmero, egress allowlist).
4. O `CapabilityEnforcer` valida o conjunto de *capabilities* do `AgentSpec` (**G1**).
5. O `McpHost` inicia e carrega o catálogo de ferramentas permitido.
6. A `ExecutionStateMachine` transiciona `boot→ready` (T1).
7. O `EventPublisher` publica `aios.<t>.agent.runtime.booted` com `cold_start_ms` medido.
8. O runtime retorna `BootReply` (`runtime_id`, `session_id`, estado `ready`).

**Fluxos alternativos:**
- FA1. Supervisor consulta `GetStatus` logo após `Boot` para confirmar `ready` antes de enviar `SubmitTask`.

**Exceções:**
- E1. `AgentSpec` inválido → `AIOS-RUNTIME-0009` (T2, `boot→failed`), nenhuma sessão utilizável.
- E2. *Capabilities* ausentes/incompatíveis → `AIOS-RUNTIME-0003`, boot abortado (G0/G1 falha).
- E3. Falha de preparação de sandbox (seccomp/cgroups/netns) → `AIOS-RUNTIME-0002`, `boot→failed`.
- E4. `Idempotency-Key` já usada com o mesmo payload → retorna a mesma `BootReply` original, sem novo `RuntimeInstance`/`AgentSession`.

**Pós-condições:**
- Q1. `AgentSession.state = ready`; `RuntimeInstance.state = busy`.
- Q2. Evento `agent.runtime.booted` publicado com `cold_start_ms ≤ 250 ms` (NFR-001).
- Q3. Span OTel de boot correlacionado por `traceparent`/`tenant`/`agent`.

---

## UC-002 — Boot de agente sem pool quente (cold boot)

**Ator primário:** Runtime Supervisor.

**Pré-condições:**
- P1. Nenhuma instância `idle` disponível no pool quente no momento da requisição.
- P2. `AgentSpec` válido.

**Fluxo principal:**
1. O Supervisor solicita `Boot` sem instância pré-aquecida disponível.
2. O Supervisor cria um novo processo runtime "a frio" (fora do escopo deste módulo — gestão de pool) e delega o boot ao `RuntimeBootstrapper` recém-iniciado.
3. Os passos 2–8 de **UC-001** se repetem, porém sem o benefício de intérprete pré-forkado (*copy-on-write*, ADR-0073).
4. O `EventPublisher` publica `agent.runtime.booted` com `cold_start_ms` medido no caminho frio.

**Fluxos alternativos:**
- FA1. Se nenhum slot puder ser criado (limite de nó/recursos), o Supervisor recebe `AIOS-RUNTIME-0001` (503, `retriable=true`) antes mesmo de alcançar este módulo.

**Exceções:**
- Mesmas de **UC-001** (E1–E3).

**Pós-condições:**
- Q1. `AgentSession.state = ready`, com `cold_start_ms ≤ 900 ms` (NFR-001, caminho frio).
- Q2. Métrica `aios_runtime_cold_start_ms` registra a amostra separadamente por *label* `warm=false` para acompanhamento de degradação sob pico.

---

## UC-003 — Execução completa de um passo ReAct

**Ator primário:** `CognitiveLoopEngine` (interno, acionado por `SubmitTask`).
**Atores secundários:** `PerceptionModule`, `ReasoningModule`, `ActionExecutor`, `ObservationCollector`, `010-Memory`, `011-Context`, `017-Model-Router`, `015-Tool-Manager`.

**Pré-condições:**
- P1. `AgentSession.state = ready` ou `thinking` (retomando o loop).
- P2. Orçamento disponível no `QuotaGovernor` (`consumed ≤ budget`).

**Fluxo principal:**
1. `ready→thinking` (T3): `PerceptionModule` monta o *percept* via `ContextClient`/`MemoryClient`.
2. `ReasoningModule` envia o *percept* ao `ModelRouterClient → 017-Model-Router`; recebe *thought* e decisão de ação.
3. Decisão = `tool_call`: `thinking→acting` (T4), guarda **G2** (capability da tool concedida via `CapabilityEnforcer`).
4. `ActionExecutor` invoca a ferramenta via `McpHost`/`ToolInvoker → 015-Tool-Manager`; `acting→waiting-io` (T7).
5. Resultado da tool chega; `ObservationCollector` normaliza em `observation`, atualiza working memory (`010-Memory`); `waiting-io→thinking` (T8).
6. O `ReActStep` completo (`percept_ref`, `thought` redigido, `action_type`, `observation`, `status=ok`, `latency_ms`) é persistido/publicado (`aios.<t>.task.execution.step`).
7. Redação de PII (`pii.redaction.enabled`) é aplicada a `thought`/`observation` antes da publicação (FR-015).

**Fluxos alternativos:**
- FA1. Decisão = `final_answer`: `thinking→done` (T5, guarda **G5**) — ver **UC-004**.
- FA2. Decisão = `replan` ou falha recuperável detectada: `thinking→reflecting` (T6) — ver **UC-005**.

**Exceções:**
- E1. Model Router indisponível → ver **UC-012**.
- E2. Tool Manager indisponível/timeout → ver **UC-013**.
- E3. Capability negada para a tool → ver **UC-011**.

**Pós-condições:**
- Q1. Exatamente um `ReActStep` novo persistido, com `index` incrementado.
- Q2. `consumed` do `QuotaGovernor` atualizado (tokens do `ModelCall`, custo do `ToolInvocation`).
- Q3. Span OTel do passo completo, com atributos `phase`, `status`, `tenant`, `agent`.

---

## UC-004 — Loop com múltiplos passos até resposta final

**Ator primário:** `CognitiveLoopEngine`.

**Pré-condições:**
- P1. `AgentSession.state = ready`.
- P2. Objetivo do agente ainda não satisfeito.

**Fluxo principal:**
1. **UC-003** se repete em sequência (`thinking→acting→waiting-io→thinking`) por N iterações, incrementando `step_cursor` a cada passo.
2. Em uma iteração, `ReasoningModule` decide `final_answer` porque o critério de sucesso do goal foi satisfeito (**G5**).
3. `thinking→done` (T5); `terminal_reason` é preenchido com o resumo do resultado.
4. `EventPublisher` publica `aios.<t>.task.execution.completed` (`status=SUCCEEDED`, `costUsd`, `tokens` agregados de todos os `ModelCall`/`ToolInvocation` da sessão).

**Fluxos alternativos:**
- FA1. Limite de passos/profundidade/fan-out atingido antes da resposta final → ver **UC-006**.

**Exceções:**
- E1. Erro não recuperável em qualquer passo → `T9`/`T11` para `failed`, `aios.<t>.task.execution.failed` publicado com `code`/`reason`/`retriable`.

**Pós-condições:**
- Q1. `AgentSession.state = done` (terminal); `step_cursor` reflete o total de passos executados.
- Q2. Todos os `ReActStep` da sessão formam uma trilha auditável e reconstruível (`025-Audit`).

---

## UC-005 — Replanejamento por falha recuperável

**Ator primário:** `CognitiveLoopEngine`, `PlanningClient → 012-Planning`.

**Pré-condições:**
- P1. Uma falha recuperável foi detectada (ex.: tool retornou erro após retries, mas dentro do orçamento) ou há deriva de objetivo.
- P2. `plan_urn` corrente existe (pode ser nulo na primeira iteração).

**Fluxo principal:**
1. `thinking→reflecting` (T6, guarda **G4**: erro recuperável/deriva de objetivo).
2. `PlanningClient` solicita replanejamento a `012-Planning`, enviando o histórico recente de `ReActStep` como contexto de falha.
3. `012-Planning` retorna um novo `plan_urn` com passos/critérios de sucesso atualizados.
4. `reflecting→thinking` (T10, guarda **G6**: plano válido recebido); `AgentSession.plan_urn` atualizado.
5. O loop retoma em **UC-003**/**UC-004** com o novo plano.

**Fluxos alternativos:**
- FA1. Reflexão sem necessidade de novo plano (mera correção de abordagem no mesmo plano): o loop retorna a `thinking` sem chamar `012-Planning` (implementação interna do `ReasoningModule`).

**Exceções:**
- E1. `012-Planning` não retorna plano viável (orçamento de replanejamento esgotado, objetivo inatingível) → `reflecting→failed` (T11, guarda **G7**), `AIOS-RUNTIME-0014`.

**Pós-condições:**
- Q1. `AgentSession.plan_urn` reflete o plano ativo mais recente.
- Q2. Evento de transição de estado publicado para cada mudança (`reflecting`, retorno a `thinking`).

---

## UC-006 — Limite de passos/profundidade/fan-out excedido

**Ator primário:** `QuotaGovernor`, `CognitiveLoopEngine`.

**Pré-condições:**
- P1. Sessão em execução ativa (`thinking`/`acting`).
- P2. Configuração `loop.max_steps`/`loop.max_recursion_depth`/`loop.max_tool_fanout` vigente para o tenant/agente.

**Fluxo principal:**
1. Antes de iniciar um novo passo (T3) ou uma nova invocação de tool em paralelo, o `QuotaGovernor` verifica os três limites.
2. Um dos limites seria excedido pela continuação.
3. O `QuotaGovernor` sinaliza rejeição ao `CognitiveLoopEngine`, que **não** executa o passo/fan-out adicional.
4. `AIOS-RUNTIME-0012` é retornado/registrado; a sessão transiciona a `failed` (ou é oferecida a `reflecting` conforme política de degradação, se configurado) via **T11**.
5. `aios.<t>.task.execution.failed` publicado com `code=AIOS-RUNTIME-0012`, `retriable=false`.

**Fluxos alternativos:**
- FA1. Limite de fan-out apenas: a invocação de tool excedente é rejeitada, mas as demais em andamento continuam; o loop pode prosseguir se ao menos uma invocação retornar sucesso (decisão de implementação documentada em `ClassDiagrams.md`).

**Exceções:**
- Nenhuma exceção adicional além da rejeição determinística descrita.

**Pós-condições:**
- Q1. Nenhum passo/invocação além do limite configurado é executado (verificável por contagem exata em teste de integração).
- Q2. Métrica `aios_runtime_step_latency_ms` não é incrementada para o passo rejeitado (ele nunca inicia).

---

## UC-007 — Esgotamento de cota local com suspensão e checkpoint

**Ator primário:** `QuotaGovernor`, `CheckpointManager`.

**Pré-condições:**
- P1. Sessão ativa consumindo tokens/USD/wall-clock/passos.
- P2. `checkpoint.enabled = true`.

**Fluxo principal:**
1. `QuotaGovernor` detecta que `consumed` atingiria/excederia `budget` (tokens, `budget.max_cost_usd`, `loop.wall_clock_budget_ms` ou `loop.max_steps`).
2. `aios.<t>.agent.runtime.quota_exceeded` é publicado com `resource`, `limit`, `consumed`.
3. Estado ativo → `suspending` (T12), acionado pelo próprio `QuotaGovernor`.
4. `CheckpointManager` serializa working memory + `step_cursor` + plano; calcula `checksum` sha256.
5. `suspending→suspended` (T13, guarda **G8**: checkpoint íntegro); `aios.<t>.agent.runtime.suspended` publicado com `checkpoint_id`.

**Fluxos alternativos:**
- FA1. `checkpoint.enabled = false` ou checkpoint impossível (ex.: recurso interno indisponível): a sessão transiciona diretamente a `terminating→failed` com `AIOS-RUNTIME-0004`.

**Exceções:**
- E1. Falha ao gerar checkpoint (integridade) → `AIOS-RUNTIME-0010`; a sessão permanece no estado ativo anterior e uma nova tentativa é agendada, ou a sessão é encerrada como `failed` se o retry também falhar.

**Pós-condições:**
- Q1. `AgentSession.state = suspended` (quiescente, retomável) na maioria dos casos com `checkpoint.enabled=true`.
- Q2. RAM da instância de runtime é liberada; `RuntimeInstance.state = idle` volta ao pool.
- Q3. Consumo consolidado (`ModelCall.cost_usd`, `ToolInvocation.cost_usd`) permanece disponível para consulta de FinOps.

---

## UC-008 — Crash do processo entre checkpoints e retomada

**Ator primário:** Runtime Supervisor (recriação), `CheckpointManager` (na nova instância).

**Pré-condições:**
- P1. Um `Checkpoint` íntegro existe para a sessão (do intervalo `checkpoint.interval_steps` anterior ao crash).
- P2. Nenhuma outra instância de runtime detém o *lease* Redis da sessão.

**Fluxo principal:**
1. O processo runtime original falha (crash) sem publicar o checkpoint mais recente pretendido; o outbox local perde-se com o processo (dados não persistidos além do último `flush` confirmado).
2. O Supervisor detecta heartbeat perdido (ver **UC-009**) e recria uma instância de runtime.
3. O Supervisor invoca `Resume(ResumeRequest)` referenciando o último `checkpoint_id` durável conhecido.
4. A nova instância adquire o *lease* distribuído (Redis) por `session_id` (**G9**, ADR-0079).
5. `resuming` (T14): `CheckpointManager` restaura `working_memory_ref`, `plan_snapshot_ref`, `step_cursor` a partir do checkpoint íntegro.
6. `resuming→(estado salvo)` (T15): a sessão retoma em `thinking` a partir do `step_cursor` restaurado.
7. `aios.<t>.agent.runtime.resumed` publicado com `session_id`, `checkpoint_id`.

**Fluxos alternativos:**
- FA1. Nenhum checkpoint existe ainda (crash antes do primeiro `checkpoint.interval_steps`): a sessão é considerada perdida e reiniciada do zero pelo control plane (fora do escopo deste módulo; decisão de `009-Scheduler`).

**Exceções:**
- E1. *Lease* já detido por outra instância (condição de corrida) → `Resume` rejeitado até liberação do lease, prevenindo dupla execução (RSK-A05).
- E2. Checkpoint corrompido (checksum inválido) → `AIOS-RUNTIME-0010`; retomada do checkpoint íntegro anterior, se existir.

**Pós-condições:**
- Q1. RTO da retomada ≤ 30 s (NFR-006); no máximo 1 passo do loop é perdido em relação ao último checkpoint confirmado (RPO ≤ 1 passo).
- Q2. Exatamente uma instância de runtime está ativa para a sessão após a retomada.

---

## UC-009 — Perda de heartbeat e recriação pelo Supervisor

**Ator primário:** Runtime Supervisor.
**Ator secundário:** `SupervisorChannel`/`HealthProbe`.

**Pré-condições:**
- P1. `heartbeat.interval_ms` configurado; Supervisor monitora `last_heartbeat_at` de cada `RuntimeInstance`.

**Fluxo principal:**
1. A instância deixa de emitir `aios.<t>.agent.runtime.heartbeat` por um intervalo além do limiar de watchdog do Supervisor.
2. O Supervisor marca a `RuntimeInstance` como `dead` (§4.3 do brief, tabela de transições).
3. Se havia uma `AgentSession` ativa associada, o Supervisor aciona o fluxo de retomada — ver **UC-008**.

**Fluxos alternativos:**
- FA1. Heartbeat atrasado mas não perdido (latência transitória de rede): o Supervisor tolera até N intervalos perdidos antes de declarar `dead` (configuração do Supervisor, fora deste módulo).

**Exceções:**
- Nenhuma exceção adicional; este UC é, em si, um fluxo de tratamento de exceção de nível de infraestrutura.

**Pós-condições:**
- Q1. `RuntimeInstance.state = dead`; recursos do nó são liberados pelo Supervisor.
- Q2. Nenhuma sessão permanece "órfã" além do RTO definido em NFR-006.

---

## UC-010 — Violação de sandbox (tentativa de escape)

**Ator primário:** `SandboxManager` (via monitor `seccomp-bpf`).

**Pré-condições:**
- P1. Sandbox ativo com `SandboxProfile` aplicado (namespaces, seccomp, cgroups, egress allowlist).

**Fluxo principal:**
1. O processo do agente tenta uma syscall fora da allowlist `seccomp` ou uma conexão de rede fora de `sandbox.egress_allowlist`.
2. O kernel do SO/`SandboxManager` intercepta e bloqueia a operação.
3. `SandboxManager` classifica a tentativa como `sandbox_violation` e aciona `kill` imediato da sessão/instância — qualquer estado ativo → `terminating` (T16).
4. `aios.<t>.agent.runtime.sandbox_violation` publicado com `violation_type`, consumido por `025-Audit`.
5. `AIOS-RUNTIME-0015` registrado; `terminating→failed` (T17).

**Fluxos alternativos:**
- Nenhum — violação de sandbox **NÃO DEVE** ter caminho de recuperação automática; é sempre tratada como incidente de segurança.

**Exceções:**
- Nenhuma adicional (este UC já é, por definição, um caminho de exceção de segurança).

**Pós-condições:**
- Q1. `AgentSession.state = failed`; `RuntimeInstance` da instância comprometida é destruída pelo Supervisor (nunca reaproveitada no pool quente).
- Q2. Evento auditável imutável registrado para investigação (RSK-A01, NFR-008: 0 escapes confirmados = nenhuma violação chega a produzir efeito fora do sandbox).

---

## UC-011 — Negação de capability pelo PEP local

**Ator primário:** `CapabilityEnforcer`.

**Pré-condições:**
- P1. `ReasoningModule` decidiu uma ação privilegiada (tool call, egress, syscall cognitiva).
- P2. A *capability* necessária não foi concedida no boot nem confirmada dinamicamente pelo PDP.

**Fluxo principal:**
1. `ActionExecutor`/`ToolInvoker` solicita ao `CapabilityEnforcer` a validação da ação antes de qualquer efeito colateral.
2. O `CapabilityEnforcer` verifica o cache local de decisão; em cache-miss, consulta o PDP via Kernel/`022-Policy`.
3. Decisão = `deny` (postura *default deny* na ausência de decisão).
4. A ação **não** é executada; `AIOS-RUNTIME-0003` é retornado ao `CognitiveLoopEngine`.
5. O passo é registrado como `ReActStep.status = denied`; o loop pode tentar replanejamento (**UC-005**) ou encerrar, conforme a gravidade.

**Fluxos alternativos:**
- FA1. Decisão via cache local (sem round-trip ao PDP), quando a *capability* já foi previamente negada/concedida e ainda não invalidada por `aios.<t>.policy.decision.updated`.

**Exceções:**
- Nenhuma adicional — a negação em si é o caminho principal deste UC.

**Pós-condições:**
- Q1. Nenhum efeito colateral (tool executada, egress realizado) ocorre para a ação negada.
- Q2. Evento de negação auditável correlacionado por `traceparent`, consumido por `025-Audit`.

---

## UC-012 — Indisponibilidade do Model Router e fallback

**Ator primário:** `ModelRouterClient`.

**Pré-condições:**
- P1. Sessão em `thinking`, aguardando decisão de inferência.
- P2. `model.fallback_enabled = true`.

**Fluxo principal:**
1. `ModelRouterClient` invoca `017-Model-Router`; a chamada falha (timeout/erro) dentro de `model.default_timeout_ms`.
2. O `ModelCall` é registrado com `status = timeout` ou `error`.
3. `ModelRouterClient` reintenta conforme política de retry; se persistir, e `model.fallback_enabled = true`, sinaliza fallback ao `017-Model-Router` (que escolhe modelo/endpoint alternativo).
4. Nova tentativa com `status = fallback` tem sucesso; o loop prossegue normalmente a partir de `thinking`.

**Fluxos alternativos:**
- FA1. `model.fallback_enabled = false`: sem fallback, a falha é tratada como recuperável e encaminhada a **UC-005** (replanejamento) ou, se persistente, a falha fatal.

**Exceções:**
- E1. Falhas persistentes mesmo com fallback → `AIOS-RUNTIME-0006`; se orçamento/retries esgotados, sessão → `failed` (T9/T11).

**Pós-condições:**
- Q1. `ModelCall.status` reflete o desfecho real (`ok`/`error`/`timeout`/`fallback`).
- Q2. Nenhuma chamada é feita diretamente a um provedor de LLM fora de `017-Model-Router` durante o fallback (FR-003 preservado).

---

## UC-013 — Indisponibilidade do Tool Manager e circuit breaker

**Ator primário:** `ToolInvoker`.

**Pré-condições:**
- P1. Sessão em `acting`, aguardando resultado de uma tool.
- P2. `tool.circuit_breaker.error_threshold` configurado.

**Fluxo principal:**
1. `ToolInvoker` invoca `015-Tool-Manager`; a chamada falha por timeout/erro repetidas vezes dentro da janela de avaliação.
2. Ao atingir `tool.circuit_breaker.error_threshold`, o *circuit breaker* abre; novas invocações à mesma dependência retornam imediatamente `result_status = circuit_open` sem nova tentativa de rede.
3. `ObservationCollector` normaliza o resultado como falha recuperável; `thinking→reflecting` é avaliado (**UC-005**).
4. Após o intervalo de meia-abertura, o breaker testa a dependência; se `015` responde normalmente, o breaker fecha (NFR-016: ≤ 5 min após recuperação).

**Fluxos alternativos:**
- FA1. Falha isolada (não atinge o limiar): apenas retry com backoff/jitter conforme `tool.retry.max_attempts`, sem abrir o breaker.

**Exceções:**
- E1. Timeout definitivo sem recuperação dentro do orçamento → `AIOS-RUNTIME-0005`; se retries esgotados e não recuperável, `waiting-io→failed` (T9, guarda **G7**).

**Pós-condições:**
- Q1. `aios_runtime_*` (métricas de circuit breaker, ver `Metrics.md`) refletem o estado `open`/`half-open`/`closed` em tempo real.
- Q2. Nenhuma invocação adicional é enviada a `015-Tool-Manager` enquanto o breaker está `open`, reduzindo carga sobre a dependência degradada.

---

## UC-014 — Repetição idempotente de `SubmitTask`

**Ator primário:** Control plane (chamador de `SubmitTask`).

**Pré-condições:**
- P1. Uma submissão anterior com a mesma `Idempotency-Key` foi processada dentro da janela de retenção (≥ 24h, RFC-0001 §5.5).

**Fluxo principal:**
1. O chamador reenvia `SubmitTask(TaskRequest)` com a mesma `Idempotency-Key` e o mesmo payload (ex.: por timeout de rede na resposta original).
2. O runtime identifica a chave já processada e retorna o mesmo `stream ExecutionEvent`/resultado da submissão original, sem criar nova `AgentSession` nem consumir cota adicional.

**Fluxos alternativos:**
- FA1. A sessão original ainda está em andamento: o runtime religa o chamador ao stream já ativo (mesmo `session_id`).

**Exceções:**
- E1. Mesma `Idempotency-Key` com payload **divergente** do original → `AIOS-RUNTIME-0007` (409), sem processar a nova submissão.

**Pós-condições:**
- Q1. Exatamente uma `AgentSession` existe para a `Idempotency-Key`, independentemente do número de repetições.
- Q2. Nenhum evento de execução duplicado é publicado além dos originais.

---

## UC-015 — Loop travado detectado pelo watchdog

**Ator primário:** `HealthProbe` (watchdog).

**Pré-condições:**
- P1. Sessão ativa em `thinking`/`acting`/`waiting-io` sem progresso observável.
- P2. `watchdog.stuck_timeout_ms` configurado.

**Fluxo principal:**
1. O watchdog detecta ausência de transição de estado/novo `ReActStep` por mais de `watchdog.stuck_timeout_ms`.
2. `AIOS-RUNTIME-0008` é registrado; o watchdog interrompe o passo em curso.
3. O watchdog tenta replanejamento (**UC-005**) como primeira alternativa (Princípio 7 — degradação graciosa antes de falha dura).
4. Se o replanejamento também travar ou não for viável, a sessão é encerrada: `terminating→failed` (T16/T17).

**Fluxos alternativos:**
- FA1. O passo era legitimamente longo (falso positivo): mitigação operacional é aumentar `watchdog.stuck_timeout_ms` para o perfil do tenant/agente (ver risco arquitetural em `Architecture.md` §10).

**Exceções:**
- Nenhuma adicional.

**Pós-condições:**
- Q1. Nenhuma sessão permanece travada indefinidamente consumindo recursos do nó.
- Q2. RTO de recuperação (via checkpoint, se aplicável) ≤ 30 s (NFR-006).

---

## UC-016 — Streaming de passos ao control plane

**Ator primário:** Control plane (ex.: WebConsole via serviço intermediário).

**Pré-condições:**
- P1. `AgentSession` ativa e conhecida pelo chamador (`session_id`).

**Fluxo principal:**
1. O chamador invoca `StreamSteps(SessionRef)`.
2. O runtime abre um `stream ReActStep` e emite cada passo assim que produzido pelo `CognitiveLoopEngine`, em ordem de `index`.
3. O stream permanece aberto até a sessão atingir estado terminal (`done`/`failed`) ou o chamador encerrar a assinatura.

**Fluxos alternativos:**
- FA1. Chamador se reconecta após queda de rede: reabre `StreamSteps` e recebe os passos a partir do `index` mais recente confirmado (semântica *at-least-once*; consumidor deduplica por `step_id`).

**Exceções:**
- E1. `session_id` inexistente/não pertencente ao tenant do chamador → erro de leitura com isolamento de tenant (ver **UC-017**).

**Pós-condições:**
- Q1. Nenhum passo é omitido na sequência observada pelo chamador (ordenação garantida por `index`).

---

## UC-017 — Isolamento de tenant (rejeição de tenant divergente)

**Ator primário:** Qualquer chamador de `aios.runtime.v1` (Supervisor, control plane).

**Pré-condições:**
- P1. O chamador está autenticado via mTLS/claims com um `tenant` definido no contexto.
- P2. A requisição referencia um recurso (`session_id`, `runtime_id`) associado a um `tenant_id` diferente do contexto autenticado.

**Fluxo principal:**
1. O chamador invoca qualquer método (`Boot`, `SubmitTask`, `GetStatus`, endpoints REST de leitura) referenciando um recurso.
2. O runtime compara `tenant_id` do recurso-alvo com o `tenant` do contexto autenticado (claim assinada / `X-AIOS-Tenant`).
3. Divergência detectada → requisição rejeitada antes de qualquer leitura/mutação do recurso.

**Fluxos alternativos:**
- Nenhum — este UC é, por definição, um caminho de rejeição.

**Exceções:**
- E1. Tenant divergente → erro 403 equivalente (mapeado no domínio `RUNTIME`; ver `API.md` §catálogo), sem vazar dados do tenant real na resposta (RFC-0001 §6).

**Pós-condições:**
- Q1. Nenhum dado de um tenant é exposto a um chamador autenticado como outro tenant (NFR-012).
- Q2. Tentativa registrada para auditoria de segurança.

---

## UC-018 — Compatibilidade de ABI entre versões major

**Ator primário:** Equipe de plataforma (teste de compatibilidade em CI/CD).

**Pré-condições:**
- P1. Duas versões major do pacote `aios.runtime.v1`/`v2` coexistem durante a janela de migração (RFC-0001 §5.7).

**Fluxo principal:**
1. A suíte de contract test executa os clientes da versão `N-1` contra uma instância de runtime que já expõe a versão `N`.
2. Campos aditivos novos da versão `N` são ignorados com segurança pelo cliente `N-1` (evolução aditiva, RFC-0001 §5.7).
3. Métodos/contratos da versão `N-1` continuam funcionalmente corretos.
4. Em paralelo, o catálogo MCP tolera adição de novas ferramentas via `aios.<t>.tool.registry.updated` sem exigir reinício da sessão ativa.

**Fluxos alternativos:**
- Nenhum — este é um teste de regressão de compatibilidade, não um fluxo operacional de agente.

**Exceções:**
- E1. Quebra de compatibilidade detectada (campo obrigatório removido/renomeado) → gate de CI bloqueia o *release* até correção ou depreciação formal coordenada.

**Pós-condições:**
- Q1. Clientes de versão `N-1` seguem operacionais durante toda a janela de coexistência de ≥ 2 majors (NFR-014).

---

## Notas de rastreabilidade

- A relação completa `UC-NNN → FR/NFR → Teste` está consolidada em
  `Requirements.md` §5; este documento é a fonte de detalhamento de fluxo
  para cada célula daquela matriz.
- Diagramas de sequência ASCII correspondentes aos fluxos felizes e de falha
  mais críticos (UC-001, UC-003, UC-005, UC-007, UC-008, UC-010, UC-012,
  UC-013, UC-014) estão em `SequenceDiagrams.md`.
