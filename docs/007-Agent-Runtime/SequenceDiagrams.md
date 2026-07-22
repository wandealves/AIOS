---
Documento: SequenceDiagrams
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0072, ADR-0073, ADR-0074, ADR-0076, ADR-0077, ADR-0078, ADR-0079
RFCs relacionados: RFC-0001, RFC-0070, RFC-0071
Depende de: ./StateMachine.md, ./ClassDiagrams.md, ./Architecture.md, ../008-Agent-Lifecycle/, ../015-Tool-Manager/, ../017-Model-Router/, ../022-Policy/
---

# 007-Agent-Runtime — Diagramas de Sequência

> Fluxos críticos do Agent Runtime, em caminho feliz e em falha, com
> participantes, mensagens síncronas (`──▶`/`◀──`, chamada+retorno) e
> assíncronas (`┄┄▶`, publicação de evento/*fire-and-forget*), e timeouts
> relevantes. Todos os fluxos respeitam a FSM de `./StateMachine.md`, as
> interfaces de `./ClassDiagrams.md` e os contratos de
> `../003-RFC/RFC-0001-Architecture-Baseline.md` (`traceparent`,
> `X-AIOS-Tenant`, `Idempotency-Key`, envelope CloudEvents, envelope de erro
> RFC 7807). Cada diagrama referencia o `UC-NNN` correspondente em
> `./UseCases.md`.

## Índice

1. Convenções de notação
2. `Boot` — caminho feliz com pool quente (UC-001)
3. Passo ReAct completo — `thinking→acting→waiting-io→thinking` (UC-003)
4. Replanejamento por falha recuperável (UC-005)
5. Esgotamento de cota → suspensão com checkpoint (UC-007)
6. Crash do processo e retomada via checkpoint (UC-008)
7. Violação de sandbox → kill imediato (UC-010)
8. Indisponibilidade do Model Router com fallback (UC-012)
9. Indisponibilidade do Tool Manager — abertura do circuit breaker (UC-013)
10. Replay idempotente de `SubmitTask` (UC-014)
11. Referências

---

## 1. Convenções de Notação

```
Participante A          Participante B
     │                        │
     │──── msg síncrona ─────▶│   chamada bloqueante (gRPC), aguarda retorno
     │◀─── retorno ───────────│
     │                        │
     │┄┄┄ msg assíncrona ┄┄┄▶│   publicação de evento (NATS/JetStream)
     │                        │
     ╎── timeout T ──╎         marcador de temporizador/deadline
```

Latências-alvo citadas referem-se às metas de `./NonFunctionalRequirements.md`
(idênticas ao `_DESIGN_BRIEF.md` §7.2).

---

## 2. `Boot` — Caminho Feliz com Pool Quente (UC-001)

**Pré-condição:** instância `idle` no pool quente; `AgentSpec` válido;
`Idempotency-Key` presente. **Pós-condição:** `AgentSession.state = ready`;
evento `agent.runtime.booted` publicado; `cold_start_ms ≤ 250 ms` (NFR-001).

```
Supervisor   SupervisorChannel  RuntimeBootstrapper  SandboxManager  CapabilityEnforcer  McpHost  ExecutionStateMachine  EventPublisher
    │               │                   │                  │                │              │              │                    │
    │──Boot(spec)──▶│                   │                  │                │              │              │                    │
    │               │──dispatch────────▶│                  │                │              │              │                    │
    │               │                   │──PrepareSandbox─▶│                │              │              │                    │
    │               │                   │◀── sandbox ready ─│                │              │              │                    │
    │               │                   │──ValidateCaps───────────────────▶│              │              │                    │
    │               │                   │◀── G1: allow ─────────────────────│              │              │                    │
    │               │                   │──StartMcpHost───────────────────────────────────▶│              │                    │
    │               │                   │◀── catálogo carregado ──────────────────────────│              │                    │
    │               │                   │──Transition(boot→ready, T1)─────────────────────────────────▶│                    │
    │               │                   │                  │                │              │              │──publish(booted)──▶│
    │               │◀── ready ─────────│                  │                │              │              │                    │┄┄▶ aios.<t>.agent.runtime.booted
    │◀─BootReply(runtime_id,session_id)─│                  │                │              │              │                    │
```

**Falhas associadas:** `AgentSpec` inválido → `AIOS-RUNTIME-0009` (T2,
`boot→failed`); capabilities ausentes → `AIOS-RUNTIME-0003` (G0/G1 falha);
falha de sandbox → `AIOS-RUNTIME-0002`.

---

## 3. Passo ReAct Completo (UC-003)

**Pré-condição:** `AgentSession.state = ready`/`thinking`; orçamento
disponível. **Pós-condição:** um `ReActStep` novo publicado; `consumed`
atualizado.

```
CognitiveLoopEngine  PerceptionModule  ContextClient  MemoryClient  ReasoningModule  ModelRouterClient  ActionExecutor  ToolInvoker  ObservationCollector
        │                   │               │             │              │                 │                │             │               │
        │──Perceive()──────▶│               │             │              │                 │                │             │               │
        │                   │──GetContext──▶│             │              │                 │                │             │               │
        │                   │◀── contexto ──│             │              │                 │                │             │               │
        │                   │──Recall──────────────────▶│              │                 │                │             │               │
        │                   │◀── memória ────────────────│              │                 │                │             │               │
        │◀── percept ───────│               │             │              │                 │                │             │               │
        │──Think(percept)──────────────────────────────────────────────▶│                 │                │             │               │
        │                   │               │             │              │──Infer(prompt)─▶│                │             │               │
        │                   │               │             │              │  ╎── timeout `model.default_timeout_ms` ──╎    │             │               │
        │                   │               │             │              │◀─ thought+ação ──│                │             │               │
        │◀── decisão=tool_call [G2] ────────────────────────────────────│                 │                │             │               │
        │──Act(tool_call)──────────────────────────────────────────────────────────────────────────────▶│             │               │
        │                   │               │             │              │                 │                │──Invoke(tool)▶│               │
        │                   │               │             │              │                 │                │  ╎── timeout `tool.invocation_timeout_ms` ──╎
        │                   │               │             │              │                 │                │◀── resultado ─│               │
        │◀── observação bruta ──────────────────────────────────────────────────────────────────────────────────────────│               │
        │──Observe(resultado)──────────────────────────────────────────────────────────────────────────────────────────────────────────▶│
        │                   │               │             │              │                 │                │             │──UpdateWM─▶│(MemoryClient)
        │◀── ReActStep{phase,status,latency_ms} ───────────────────────────────────────────────────────────────────────────────────────│
        │┄┄ publish(task.execution.step) ┄┄▶ EventPublisher
```

**Notas:** `thought`/`observation` passam por redação de PII (FR-015) antes
da publicação. Transições de estado (`ready→thinking`, `thinking→acting`,
`acting→waiting-io`, `waiting-io→thinking`) seguem T3/T4/T7/T8 de
`./StateMachine.md`.

---

## 4. Replanejamento por Falha Recuperável (UC-005)

**Pré-condição:** falha recuperável ou deriva de objetivo detectada (guarda
**G4**). **Pós-condição:** `plan_urn` atualizado ou sessão → `failed`
(**G7**).

```
CognitiveLoopEngine  ExecutionStateMachine  PlanningClient  012-Planning  EventPublisher
        │                     │                    │              │             │
        │──falha recuperável [G4]───────────────▶│                    │              │             │
        │                     │──Transition(thinking→reflecting, T6)─│              │             │
        │                     │┄┄ publish(state=reflecting) ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄▶│
        │──Replan(histórico_falha)──────────────────────────────▶│              │             │
        │                     │                    │              │◀── decompõe ──│             │
        │◀── novo plan_urn [G6] ─────────────────────────────────│              │             │
        │                     │──Transition(reflecting→thinking, T10)│              │             │
        │                     │┄┄ publish(state=thinking, plan_urn atualizado) ┄┄▶│
        │── loop retoma (ver §3) ──▶│                    │              │             │
```

**Falha alternativa (sem plano viável):**

```
        │──Replan(histórico_falha)──────────────────────────────▶│              │             │
        │◀── nenhum plano viável / orçamento esgotado [G7] ──────│              │             │
        │                     │──Transition(reflecting→failed, T11)──────────────────────────▶│
        │                     │┄┄ publish(task.execution.failed, code=AIOS-RUNTIME-0014) ┄┄┄▶│
```

---

## 5. Esgotamento de Cota → Suspensão com Checkpoint (UC-007)

**Pré-condição:** `consumed` se aproxima/excede `budget`.
**Pós-condição:** `AgentSession.state = suspended`; RAM liberada.

```
QuotaGovernor  ExecutionStateMachine  CheckpointManager  MinIO(via control plane)  EventPublisher
      │                 │                    │                    │                     │
      │──cota excedida ──────────────────▶│                    │                     │
      │                 │──Transition(*→suspending, T12)─────────────────────────────▶│
      │                 │                    │                    │                     │┄┄▶ quota_exceeded
      │                 │──Checkpoint(session)──────────────────▶│                    │
      │                 │                    │──PutObject(blob)──▶│                     │
      │                 │                    │◀── checksum ok ────│                     │
      │                 │◀── Checkpoint{checkpoint_id} [G8] ─────│                    │
      │                 │──Transition(suspending→suspended, T13)──────────────────────▶│
      │                 │                    │                    │                     │┄┄▶ agent.runtime.suspended
```

**Falha alternativa (checkpoint corrompido):** ver §"Falhas de checkpoint" em
`./FailureRecovery.md`; a sessão é mantida no checkpoint íntegro anterior e
`AIOS-RUNTIME-0010` é reportado.

---

## 6. Crash do Processo e Retomada via Checkpoint (UC-008)

**Pré-condição:** checkpoint íntegro anterior existe; instância original
falhou sem publicar heartbeat. **Pós-condição:** sessão retomada em ≤ 30 s
(NFR-006), exatamente uma instância ativa.

```
RuntimeInstance(A, morta)   Supervisor   RuntimeInstance(B, nova)  Redis(lease)  CheckpointManager  MinIO
         ╳ (crash)              │                  │                    │                │             │
                                 │──detecta heartbeat perdido──▶│                    │                │             │
                                 │──cria/recruta instância B ──▶│                    │                │             │
                                 │──Resume(checkpoint_id)──────▶│                    │                │             │
                                 │                  │──AcquireLease(session_id)──────▶│                │             │
                                 │                  │◀── lease concedido [G9] ───────│                │             │
                                 │                  │──RestoreCheckpoint()───────────────────────────▶│             │
                                 │                  │                    │                │──GetObject──▶│
                                 │                  │                    │                │◀── blob ─────│
                                 │                  │◀── working_memory + step_cursor ──────────────────│             │
                                 │                  │──Transition(resuming→estado salvo, T15)──▶
                                 │                  │┄┄ publish(agent.runtime.resumed) ┄┄▶
                                 │◀── ResumeReply ──│                    │                │             │
```

**Condição de corrida (mitigada):** se a instância A "ressuscitasse"
tardiamente e tentasse operar a mesma sessão, a tentativa de
`AcquireLease` falharia (lease já detido por B) — nenhuma dupla execução
(RSK-A05, ADR-0079).

---

## 7. Violação de Sandbox → Kill Imediato (UC-010)

**Pré-condição:** sandbox ativo com `seccomp-bpf`/egress allowlist.
**Pós-condição:** sessão `failed`; instância destruída; evento auditável
publicado.

```
Processo do agente   SandboxManager (monitor seccomp)  ExecutionStateMachine  EventPublisher  025-Audit
        │                        │                              │                  │              │
        │──syscall proibida──────▶│                              │                  │              │
        │◀── bloqueada (SIGSYS) ──│                              │                  │              │
        │                        │──classifica: sandbox_violation ─▶│                  │              │
        │                        │──Transition(*→terminating, T16)──────────────────▶│              │
        │                        │┄┄ publish(sandbox_violation, AIOS-RUNTIME-0015) ┄┄▶│──consome────▶│
        │                        │──Transition(terminating→failed, T17)─────────────▶│              │
        │                        │──kill(processo)──────────────▶│                  │              │
```

**Invariante:** não há caminho de recuperação automática a partir deste
fluxo — toda violação é tratada como incidente de segurança (ver
`./Security.md`).

---

## 8. Indisponibilidade do Model Router com Fallback (UC-012)

**Pré-condição:** `model.fallback_enabled = true`. **Pós-condição:** loop
prossegue via modelo alternativo, ou sessão degrada para replanejamento/falha.

```
ReasoningModule   ModelRouterClient   017-Model-Router (primário)   017-Model-Router (fallback)
       │                  │                     │                             │
       │──Think(percept)──▶│                     │                             │
       │                  │──Infer()────────────▶│                             │
       │                  │  ╎── timeout `model.default_timeout_ms` ──╎        │
       │                  │◀── timeout/erro ─────│                             │
       │                  │──retry (backoff+jitter)──────────────────────────▶│
       │                  │◀── ainda falha ───────────────────────────────────│
       │                  │──solicita fallback──────────────────────────────▶│
       │                  │◀── thought+ação (status=fallback) ────────────────│
       │◀── decisão de ação ──│                     │                             │
```

**Falha persistente:** se o fallback também falhar, `AIOS-RUNTIME-0006` é
reportado; se orçamento/retries esgotados, a sessão segue para
replanejamento (§4) ou `waiting-io→failed` (T9).

---

## 9. Indisponibilidade do Tool Manager — Abertura do Circuit Breaker (UC-013)

**Pré-condição:** falhas repetidas de `015-Tool-Manager` dentro da janela de
avaliação. **Pós-condição:** breaker `open`; novas chamadas rejeitadas
localmente até meia-abertura.

```
ActionExecutor   ToolInvoker (circuit breaker)   015-Tool-Manager
      │                    │                            │
      │──Invoke(tool)─────▶│                            │
      │                    │──gRPC call────────────────▶│
      │                    │  ╎── timeout `tool.invocation_timeout_ms` ──╎
      │                    │◀── timeout ─────────────────│
      │                    │  (repete até tool.circuit_breaker.error_threshold)
      │                    │──breaker: CLOSED→OPEN───────│
      │──Invoke(tool) [novo]▶│                            │
      │◀── result_status=circuit_open (sem chamada de rede) │
      │            ... após período de meia-abertura ...
      │                    │──probe (half-open)─────────▶│
      │                    │◀── sucesso ──────────────────│
      │                    │──breaker: HALF_OPEN→CLOSED──│
```

**Meta de recuperação (NFR-016):** o breaker DEVE fechar em ≤ 5 min após a
dependência voltar a responder corretamente.

---

## 10. Replay Idempotente de `SubmitTask` (UC-014)

**Pré-condição:** submissão original processada com sucesso dentro da janela
de retenção da `Idempotency-Key` (≥ 24h). **Pós-condição:** nenhum efeito
colateral duplicado.

```
Control plane   RuntimeControl/AgentExecution   IdempotencyCache (local)   AgentSession existente
      │                       │                          │                        │
      │──SubmitTask(key=K, payload=P) [original]────────▶│                        │
      │                       │──miss────────────────────▶│                        │
      │                       │──cria AgentSession─────────────────────────────────▶│
      │◀── stream ExecutionEvent ─────────────────────────────────────────────────│
      │  ... (rede falha antes da confirmação ao chamador) ...
      │──SubmitTask(key=K, payload=P) [repetição]────────▶│                        │
      │                       │──hit(K,P)─────────────────▶│                        │
      │                       │──religa ao stream ativo/resultado já produzido ────▶│
      │◀── mesmo stream/resultado (sem nova sessão) ──────────────────────────────│
```

**Divergência de payload:** se a repetição usar a mesma `key=K` com `payload
≠ P`, o runtime retorna `AIOS-RUNTIME-0007` (409) imediatamente, sem tocar a
sessão original.

---

## 11. Referências

- Máquina de estados referenciada em todas as transições: `./StateMachine.md`.
- Interfaces e contratos de classe dos participantes: `./ClassDiagrams.md`.
- Contratos gRPC formais (`Boot`, `SubmitTask`, `Suspend`, `Resume`, `Kill`,
  `Drain`, `GetStatus`, `StreamSteps`): `./API.md`.
- Catálogo de eventos publicados: `./Events.md`.
- Casos de uso detalhados (ator/pré/pós-condição completos): `./UseCases.md`.
- Contratos centrais (correlação, idempotência, envelope): `../003-RFC/RFC-0001-Architecture-Baseline.md`.
