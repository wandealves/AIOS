---
Documento: Metrics
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0010 (global), ADR-0077
RFCs relacionados: RFC-0001
Depende de: ../024-Observability/, ./NonFunctionalRequirements.md
---

# 007-Agent-Runtime — Metrics

> Catálogo de métricas OpenTelemetry/Prometheus emitidas pelo `TelemetryEmitter`.
> Nomes seguem `aios_<subsistema>_<nome>_<unidade>` (subsistema = `runtime`).
> Toda métrica carrega, no mínimo, os *labels* `tenant`, e, quando aplicável,
> `agent`/`session_id` (amostrado, nunca *high-cardinality* sem
> `telemetry.sampling_ratio` apropriado — ver `./Configuration.md`).

## Índice

1. Convenções
2. Métricas de cold start e boot
3. Métricas de loop cognitivo
4. Métricas de cota e orçamento
5. Métricas de tools e modelo
6. Métricas de eventos e outbox
7. Métricas de sandbox e segurança
8. Métricas de disponibilidade e pool
9. Referências

---

## 1. Convenções

| Aspecto | Regra |
|---------|-------|
| Prefixo | `aios_runtime_` |
| Tipos | `counter` (contagem monotônica), `gauge` (valor instantâneo), `histogram` (distribuição, com `_ms`/`_mib` etc.). |
| Labels obrigatórios | `tenant`. Onde aplicável: `phase`, `status`, `resource`, `verb`, `state`. |
| Unidade no nome | `_ms` (milissegundos), `_mib` (mebibytes), `_ratio` (0.0–1.0), `_total` (contador cumulativo), sem sufixo para contagens instantâneas. |
| Cardinalidade | `session_id`/`agent_urn` **NÃO DEVEM** ser labels de métrica (alta cardinalidade); use `trace_id` em spans/logs para granularidade por sessão. |

---

## 2. Métricas de Cold Start e Boot

| Métrica | Tipo | Labels | Descrição | NFR relacionado |
|---------|------|--------|-------------|-------------------|
| `aios_runtime_cold_start_ms` | histogram | `tenant`, `warm` (bool) | Latência de boot→ready. | NFR-001 |
| `aios_runtime_boot_failures_total` | counter | `tenant`, `code` | Falhas de boot por código de erro (`AIOS-RUNTIME-0002`/`0003`/`0009`). | NFR-005 |
| `aios_runtime_boot_capabilities_denied_total` | counter | `tenant` | Boots rejeitados por capability inválida (G1 falha). | NFR-008 |

---

## 3. Métricas de Loop Cognitivo

| Métrica | Tipo | Labels | Descrição | NFR relacionado |
|---------|------|--------|-------------|-------------------|
| `aios_runtime_step_latency_ms` | histogram | `tenant`, `phase` | Latência de passo do loop, excluindo tempo de LLM/tool. | NFR-003 |
| `aios_runtime_steps_total` | counter | `tenant`, `phase`, `status` | Total de passos ReAct executados. | FR-002 |
| `aios_runtime_replans_total` | counter | `tenant`, `outcome` (`plan_received`/`no_plan_viable`) | Total de replanejamentos acionados. | FR-006 |
| `aios_runtime_watchdog_triggers_total` | counter | `tenant` | Loops travados detectados pelo *watchdog*. | FR-016 |
| `aios_runtime_step_limit_exceeded_total` | counter | `tenant`, `dimension` (`steps`/`depth`/`fanout`) | Sessões que atingiram limite de passos/profundidade/fan-out. | FR-017 |
| `aios_runtime_sandbox_overhead_ratio` | gauge | `tenant` | Overhead de CPU do sandbox vs. baseline sem isolamento. | NFR-002 |

---

## 4. Métricas de Cota e Orçamento

| Métrica | Tipo | Labels | Descrição | NFR relacionado |
|---------|------|--------|-------------|-------------------|
| `aios_runtime_quota_exceeded_total` | counter | `tenant`, `resource` (`tokens`/`cost_usd`/`wall_clock`/`steps`) | Esgotamentos de cota local. | FR-010 |
| `aios_runtime_budget_consumed_ratio` | gauge | `tenant`, `resource` | Percentual consumido do orçamento corrente (amostrado por sessão ativa, agregado). | FR-010 |
| `aios_runtime_session_rss_mib` | gauge | `tenant` | RAM residente por sessão ativa/ociosa. | NFR-010 |

---

## 5. Métricas de Tools e Modelo

| Métrica | Tipo | Labels | Descrição | NFR relacionado |
|---------|------|--------|-------------|-------------------|
| `aios_runtime_tool_invocations_total` | counter | `tenant`, `result_status` | Total de invocações de ferramenta via `ToolInvoker`. | FR-004 |
| `aios_runtime_tool_invocation_latency_ms` | histogram | `tenant` | Latência de invocação de tool (fim-a-fim, via `015`). | NFR-003 (excluído do passo) |
| `aios_runtime_tool_circuit_breaker_state` | gauge | `tenant` (valor: 0=closed,1=half_open,2=open) | Estado corrente do *circuit breaker* de tool. | NFR-016 |
| `aios_runtime_model_calls_total` | counter | `tenant`, `status` | Total de chamadas de inferência via `ModelRouterClient`. | FR-003 |
| `aios_runtime_model_call_latency_ms` | histogram | `tenant` | Latência de inferência fim-a-fim (via `017`). | NFR-003 (excluído do passo) |
| `aios_runtime_model_fallback_total` | counter | `tenant` | Chamadas que exigiram fallback de modelo. | FR-014 |
| `aios_runtime_tokens_consumed_total` | counter | `tenant` | Tokens consumidos (prompt+completion). | NFR-010 |
| `aios_runtime_cost_usd_total` | counter | `tenant` | Custo acumulado (`ModelCall.cost_usd` + `ToolInvocation.cost_usd`). | NFR-010 |

---

## 6. Métricas de Eventos e Outbox

| Métrica | Tipo | Labels | Descrição | NFR relacionado |
|---------|------|--------|-------------|-------------------|
| `aios_runtime_events_published_total` | counter | `tenant`, `event_type` | Total de eventos publicados com sucesso. | NFR-007 |
| `aios_runtime_events_outbox_pending` | gauge | `tenant` | Eventos pendentes de publicação no outbox local. | NFR-006 (RPO) |
| `aios_runtime_events_outbox_flush_latency_ms` | histogram | `tenant` | Latência entre gravação no outbox e publicação confirmada. | NFR-007 |
| `aios_runtime_events_outbox_retries_total` | counter | `tenant` | Tentativas de republicação por falha transitória de rede/NATS. | NFR-005 |

---

## 7. Métricas de Sandbox e Segurança

| Métrica | Tipo | Labels | Descrição | NFR relacionado |
|---------|------|--------|-------------|-------------------|
| `aios_runtime_sandbox_violations_total` | counter | `tenant`, `violation_type` | Violações de sandbox detectadas (kill imediato). | NFR-008 |
| `aios_runtime_capability_denials_total` | counter | `tenant`, `action` | Ações negadas pelo PEP local (`CapabilityEnforcer`). | NFR-008 |
| `aios_runtime_pii_redactions_total` | counter | `tenant`, `field` (`thought`/`observation`) | Campos redigidos por PII antes da publicação. | NFR-013 |

---

## 8. Métricas de Disponibilidade e Pool

| Métrica | Tipo | Labels | Descrição | NFR relacionado |
|---------|------|--------|-------------|-------------------|
| `aios_runtime_availability_ratio` | gauge | `tenant` | Disponibilidade do pool (uptime observado / janela). | NFR-005 |
| `aios_runtime_sessions_active` | gauge | `tenant`, `state` (`AgentExecState`) | Sessões ativas por estado. | NFR-004 |
| `aios_runtime_instances` | gauge | `tenant`, `state` (`RuntimeState`) | Instâncias de runtime por estado (`warming`/`idle`/`busy`/`draining`/`dead`). | NFR-004 |
| `aios_runtime_resume_latency_ms` | histogram | `tenant` | Latência de retomada (`Resume` até estado ativo restaurado). | NFR-006 |
| `aios_runtime_checkpoint_integrity_failures_total` | counter | `tenant` | Falhas de checksum ao restaurar checkpoint. | NFR-006 |
| `aios_runtime_lease_conflicts_total` | counter | `tenant` | Tentativas de `Resume` que falharam por *lease* já detido (proteção contra dupla execução). | NFR-005 |

---

## 9. Referências

- Metas de SLO/SLI que cada métrica verifica: `./NonFunctionalRequirements.md`.
- Dashboards e alertas construídos sobre este catálogo: `./Monitoring.md`.
- Padrão de correlação em logs: `./Logging.md`.
- Stack de observabilidade: `../024-Observability/`.
