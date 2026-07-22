---
Documento: StateMachine
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0073, ADR-0074, ADR-0077, ADR-0079
RFCs relacionados: RFC-0001, RFC-0070
Depende de: ./_DESIGN_BRIEF.md, ../008-Agent-Lifecycle/, ../009-Scheduler/
---

# 007-Agent-Runtime — Máquina(s) de Estados

> Este módulo possui **duas** máquinas de estado canônicas, ambas definidas
> em `./_DESIGN_BRIEF.md` §4 e reproduzidas aqui sem alteração: (1) a FSM de
> **execução do agente** (`AgentExecState`), fonte da verdade local de uma
> `AgentSession`; e (2) a FSM de **instância de runtime** (`RuntimeState`),
> que descreve o ciclo de vida de um processo dentro do pool gerido pelo
> Runtime Supervisor. Nenhuma das duas é redefinida — apenas detalhada com
> ações de entrada/saída e exemplos.

## Índice

1. FSM de execução do agente (`AgentExecState`) — estados
2. FSM de execução do agente — transições, gatilhos e guardas
3. FSM de execução do agente — diagrama ASCII
4. Invariantes da FSM de execução
5. Ações de entrada/saída por estado
6. FSM de instância de runtime (`RuntimeState`)
7. Interação entre as duas FSMs
8. Referências

---

## 1. FSM de Execução do Agente (`AgentExecState`) — Estados

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `boot` | `AgentSpec` carregado; sandbox em preparação; capabilities em validação. | não |
| `ready` | Agente materializado, sandbox e MCP host prontos; aguardando início de passo. | não |
| `thinking` | `ReasoningModule` decidindo a próxima ação a partir do *percept* corrente. | não |
| `acting` | Ação decidida sendo despachada (tool call ou resposta final em preparo). | não |
| `waiting-io` | Aguardando resultado assíncrono de tool/LLM em andamento. | não |
| `reflecting` | Falha recuperável/deriva de objetivo detectada; replanejamento em curso. | não |
| `done` | Objetivo cumprido; resposta final produzida. | **sim** |
| `failed` | Erro não recuperável; sessão encerrada sem sucesso. | **sim** |
| `suspending` | Checkpoint em geração para hibernação/preempção. | não |
| `suspended` | Sessão quiescente, checkpointada; **retomável**, sem RAM ativa. | não (quiescente) |
| `resuming` | Checkpoint sendo restaurado após `resume`. | não |
| `terminating` | Encerramento forçado (`kill`/`drain`/cota fatal) em curso. | não |

**Notas:** `done` e `failed` são estados **terminais** (nenhuma transição de
saída existe). `suspended` é **quiescente**: não avança sozinho, mas não é
terminal — só sai por `resume` (→ `resuming`) ou por `kill`/`drain`
(→ `terminating`).

---

## 2. FSM de Execução do Agente — Transições, Gatilhos e Guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |
|---|-----------|---------|-------------------------------|
| T1 | `boot` → `ready` | Sandbox pronto + `AgentSpec` carregado | **G1**: *capabilities* válidas; cotas alocadas; MCP host ativo. |
| T2 | `boot` → `failed` | Erro de inicialização | **G0**: falha de sandbox/spec/capabilities. |
| T3 | `ready` → `thinking` | Início de passo do loop | Orçamento disponível (`QuotaGovernor.check_budget()` não `exhausted`). |
| T4 | `thinking` → `acting` | Decisão = `tool_call` | **G2**: *capability* da tool concedida (PEP, `CapabilityEnforcer`). |
| T5 | `thinking` → `done` | Decisão = `final_answer` | **G5**: critério de sucesso do goal satisfeito. |
| T6 | `thinking` → `reflecting` | Decisão = `replan` ou falha detectada | **G4**: erro recuperável ∨ deriva de objetivo. |
| T7 | `acting` → `waiting-io` | Tool/LLM despachado (assíncrono) | Invocação despachada, aguardando resultado. |
| T8 | `waiting-io` → `thinking` | Resultado chega | Resultado normalizado por `ObservationCollector`. |
| T9 | `waiting-io` → `failed` | Timeout/erro fatal da ação | **G7**: retries esgotados e não recuperável. |
| T10 | `reflecting` → `thinking` | Novo plano de `012-Planning` | **G6**: plano válido recebido. |
| T11 | `reflecting` → `failed` | Replanejamento impossível | **G7**: sem plano viável ∨ orçamento esgotado. |
| T12 | qualquer estado ativo → `suspending` | Sinal `suspend`/preempção/cota | Comando do Supervisor ∨ `QuotaGovernor` sinalizando esgotamento. |
| T13 | `suspending` → `suspended` | Checkpoint concluído | **G8**: checkpoint íntegro (`checksum` sha256 válido). |
| T14 | `suspended` → `resuming` | Sinal `resume` | **G9**: slot disponível ∧ checkpoint recuperável ∧ *lease* Redis adquirido. |
| T15 | `resuming` → (estado salvo no checkpoint) | Restauração completa | Working memory + `step_cursor` restaurados. |
| T16 | qualquer estado → `terminating` | `kill`/`drain`/cota fatal | Comando do Supervisor ∨ violação de sandbox. |
| T17 | `terminating` → `done`/`failed` | Encerramento limpo/sujo | Flush do outbox local + liberação de recursos de sandbox. |

**Estados ativos** (para T12/T16): qualquer estado que não seja `done`,
`failed`, `suspended` (já quiescente) ou já em `terminating`.

---

## 3. FSM de Execução do Agente — Diagrama ASCII

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

---

## 4. Invariantes da FSM de Execução

| # | Invariante |
|---|------------|
| I1 | Toda transição publica um evento (`./Events.md`) e emite um span OTel correlacionado por `traceparent`/`tenant`/`agent`. |
| I2 | `consumed ≤ budget` DEVE ser mantido a cada passo; violação força transição a `suspending` (com checkpoint) ou `terminating` (sem checkpoint viável). |
| I3 | De `suspended` só se sai por `resume` (T14) ou `terminating` (T16) — nunca automaticamente. |
| I4 | Estados terminais (`done`, `failed`) **NÃO** transicionam para nenhum outro estado. |
| I5 | `step_cursor` só é incrementado por uma transição completa de passo (`thinking→acting→waiting-io→thinking`), nunca por uma transição parcial/interrompida. |
| I6 | Uma `AgentSession` DEVE ter, no máximo, uma instância de runtime processando transições sobre ela em um dado instante (garantido pelo *lease* Redis por `session_id`, ver `Scalability.md` §"Locks"). |

---

## 5. Ações de Entrada/Saída por Estado

| Estado | Ação de entrada | Ação de saída |
|--------|-------------------|------------------|
| `boot` | `SandboxManager.prepare()`; `CapabilityEnforcer.load_boot_capabilities()`. | — |
| `ready` | `EventPublisher.publish(agent.runtime.booted)` (na primeira entrada). | — |
| `thinking` | `PerceptionModule.perceive()` monta o *percept*. | `ReasoningModule` já produziu uma decisão de ação. |
| `acting` | `ActionExecutor.dispatch()` despacha tool call ou prepara resposta final. | — |
| `waiting-io` | Inicia temporizador `tool.invocation_timeout_ms`/`model.default_timeout_ms`. | `ObservationCollector.observe()` normaliza o resultado. |
| `reflecting` | `PlanningClient.replan()` é invocado com o histórico de falha. | `AgentSession.plan_urn` atualizado (sucesso) ou motivo de falha registrado. |
| `done` | `EventPublisher.publish(task.execution.completed)`; libera *lease* Redis. | — (terminal) |
| `failed` | `EventPublisher.publish(task.execution.failed)`; libera *lease* Redis. | — (terminal) |
| `suspending` | `CheckpointManager.create()` inicia serialização. | — |
| `suspended` | `EventPublisher.publish(agent.runtime.suspended)`; `RuntimeInstance` retorna a `idle`. | `CheckpointManager.restore()` inicia. |
| `resuming` | `AcquireLease(session_id)` (Redis, ADR-0079). | `EventPublisher.publish(agent.runtime.resumed)`. |
| `terminating` | Interrompe passo em curso; drena outbox local. | Libera recursos de sandbox; `EventPublisher.publish(agent.runtime.terminated)`. |

---

## 6. FSM de Instância de Runtime (`RuntimeState`)

Reproduzida do `_DESIGN_BRIEF.md` §4.3 — descreve o ciclo de vida do
**processo** (não da sessão de agente), gerido pelo Runtime Supervisor.

```
 warming ──ready──▶ idle ──atribui agente──▶ busy ──libera──▶ idle
    │                 │                         │ drain          │ drain
    │                 └──────────── draining ◀──┴────────────────┘
    └──erro──▶ dead ◀──── heartbeat perdido / violação de sandbox ────
```

| De | Para | Gatilho | Guarda |
|----|------|---------|--------|
| `warming` | `idle` | Processo pronto, entra no pool quente | `HealthProbe` OK. |
| `idle` | `busy` | Supervisor atribui `AgentSession` | Slot livre. |
| `busy` | `idle` | Sessão terminou (`done`/`failed`/`suspended`) | Recursos liberados. |
| `idle`/`busy` | `draining` | Comando `Drain` (rollout/scale-down) | — |
| `draining` | `dead` | Sessões drenadas | Sem sessões ativas. |
| qualquer | `dead` | Heartbeat perdido/violação de sandbox | Watchdog do Supervisor. |

---

## 7. Interação Entre as Duas FSMs

Uma `RuntimeInstance` em `busy` hospeda **no máximo uma** `AgentSession`
ativa por vez (isolamento forte, um agente por processo). A relação entre as
duas FSMs é:

```
RuntimeState:     idle ────────▶ busy ─────────────────────────▶ idle
                              (atribuição)                    (liberação)
                                  │                                ▲
                                  ▼                                │
AgentExecState:  boot → ready → thinking ⇄ acting ⇄ waiting-io   done/failed
                                  │  ⇅ reflecting                   ▲
                                  ▼                                 │
                              suspending → suspended ──(resume noutra RuntimeInstance)
```

- `RuntimeInstance.state = busy` é definido no momento em que uma
  `AgentSession` entra em `boot`, e retorna a `idle` quando a sessão atinge
  `done`, `failed`, ou `suspended` (a instância é devolvida ao pool quente;
  a sessão pode ser retomada por **outra** instância `idle` posteriormente).
- Uma sessão `suspended` **não** mantém sua `RuntimeInstance` original
  ocupada — isso é o que torna o modelo *cold-storable* (V-R1 da Visão
  Global, ver `./Vision.md` §6).
- `RuntimeInstance.state = dead` força qualquer `AgentSession` associada e
  ainda ativa (não `suspended`) a ser tratada pelo Supervisor como candidata
  a retomada via checkpoint em outra instância (ver `./FailureRecovery.md`,
  modo de falha "Crash do processo runtime").

---

## 8. Referências

- Fonte canônica das duas FSMs: `./_DESIGN_BRIEF.md` §4.
- Diagramas de sequência que exercitam as transições: `./SequenceDiagrams.md`.
- Estruturas de dados e invariantes de classe: `./ClassDiagrams.md` §5.
- Modos de falha por transição: `./FailureRecovery.md`.
- Modelo de escala e *lease* de sessão: `./Scalability.md`.
