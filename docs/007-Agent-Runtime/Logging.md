---
Documento: Logging
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0010 (global)
RFCs relacionados: RFC-0001
Depende de: ../024-Observability/, ./Security.md
---

# 007-Agent-Runtime — Logging

> Convenção de log estruturado do Agent Runtime. Ainda que o control plane
> do AIOS use Serilog→Seq (.NET), o runtime (Python) emite logs
> **estruturados em JSON** compatíveis com o mesmo pipeline (via OTel
> Collector → Seq), preservando os mesmos campos de correlação exigidos por
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6.

## Índice

1. Formato e campos obrigatórios
2. Níveis de log
3. Eventos de log por componente
4. Correlação
5. Redação de PII em logs
6. Retenção
7. Exemplo de linha de log
8. Referências

---

## 1. Formato e Campos Obrigatórios

Todo log é uma linha JSON (`ndjson`) com, no mínimo:

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|---------------|-------------|
| `timestamp` | string (RFC 3339, UTC) | sim | Momento de emissão. |
| `level` | string | sim | `DEBUG`\|`INFO`\|`WARN`\|`ERROR`\|`CRITICAL` (ver §2). |
| `message` | string | sim | Mensagem legível (sem PII, sem segredo). |
| `component` | string | sim | Componente emissor (`PascalCase`, ex.: `CognitiveLoopEngine`). |
| `tenant_id` | string | sim | Fronteira de isolamento (RFC-0001 §5.1). |
| `trace_id` | string | sim (exceto boot pré-trace) | Extraído de `traceparent`. |
| `span_id` | string | sim (exceto boot pré-trace) | Span corrente. |
| `session_id` | string | quando aplicável | Sessão do agente em execução. |
| `runtime_id` | string | sim | Instância de runtime emissora. |
| `agent_urn` | string | quando aplicável | Agente materializado. |
| `event` | string | quando aplicável | Nome semântico do evento de log (ver §3), distinto do `message` livre. |
| `error_code` | string | em erros | `AIOS-RUNTIME-<NNNN>`, quando aplicável. |

---

## 2. Níveis de Log

| Nível | Uso |
|-------|-----|
| `DEBUG` | Detalhe de desenvolvimento (percept montado, payload de tool antes de redação) — **desabilitado por padrão em produção**. |
| `INFO` | Transições de estado, boots, checkpoints bem-sucedidos, heartbeats amostrados. |
| `WARN` | Retry de tool/modelo, cache-miss de capability, aproximação de limite de cota (ex.: 80% do orçamento). |
| `ERROR` | Falha recuperável tratada (timeout de tool com retry esgotado antes de replanejamento, falha de checkpoint com fallback ao anterior). |
| `CRITICAL` | Violação de sandbox, falha não recuperável que encerra a sessão, corrupção de checkpoint sem fallback disponível. |

---

## 3. Eventos de Log por Componente

| Componente | `event` | Nível | Campos adicionais |
|-------------|---------|-------|----------------------|
| `RuntimeBootstrapper` | `boot.started` / `boot.completed` / `boot.failed` | INFO/INFO/ERROR | `cold_start_ms`, `agent_spec_version` |
| `SandboxManager` | `sandbox.prepared` / `sandbox.violation` | INFO/CRITICAL | `sandbox_profile_id`, `violation_type` |
| `ExecutionStateMachine` | `state.transition` | INFO | `from_state`, `to_state`, `trigger` |
| `CognitiveLoopEngine` | `step.completed` / `step.failed` | INFO/ERROR | `step_index`, `phase`, `latency_ms` |
| `QuotaGovernor` | `quota.warning` / `quota.exceeded` | WARN/ERROR | `resource`, `limit`, `consumed` |
| `CapabilityEnforcer` | `capability.denied` / `capability.cache_invalidated` | WARN/DEBUG | `action`, `resource`, `source` (`cache`\|`pdp`) |
| `ToolInvoker` | `tool.retry` / `tool.circuit_open` / `tool.circuit_closed` | WARN/WARN/INFO | `tool_urn`, `attempt`, `error_code` |
| `ModelRouterClient` | `model.retry` / `model.fallback` | WARN/WARN | `requested_model`, `resolved_model_urn` |
| `CheckpointManager` | `checkpoint.created` / `checkpoint.restore_failed` | INFO/CRITICAL | `checkpoint_id`, `checksum_valid` |
| `HealthProbe` | `watchdog.triggered` | CRITICAL | `stuck_since_ms` |
| `EventPublisher` | `outbox.flush_failed` | ERROR | `retry_count`, `pending_count` |
| `SupervisorChannel` | `heartbeat.sent` | DEBUG (amostrado) | `state`, `consumed_summary` |

---

## 4. Correlação

- Todo log **DEVE** incluir `trace_id`/`span_id` extraídos do `traceparent`
  corrente (RFC-0001 §5.6), exceto logs anteriores ao recebimento do
  primeiro contexto de trace (ex.: inicialização do processo antes do
  primeiro `Boot`).
- `tenant_id` **DEVE** estar presente em 100% dos logs associados a uma
  sessão/instância — logs de infraestrutura pura do processo (ex.:
  inicialização do event loop) podem omiti-lo antes da primeira atribuição
  de tenant.
- Logs e spans OTel compartilham os mesmos identificadores, permitindo
  pivô direto entre "ver o log" e "ver o trace completo" no Grafana/Seq
  (`../024-Observability/`).

---

## 5. Redação de PII em Logs

- Campos `thought`, `observation`, argumentos de tool (`arguments_ref`
  resolvido) **NÃO DEVEM** aparecer em texto livre de log — apenas
  referências opacas (`step_id`, `tool_invocation_id`) que permitem
  consulta posterior via `010-Memory`/`025-Audit` sob controle de acesso
  apropriado.
- `pii.redaction.enabled=true` (default) aplica a mesma política de
  redação usada em eventos (`./Events.md`) também a mensagens de log,
  antes da serialização.
- Segredos (tokens, credenciais) **NUNCA** aparecem em log, mesmo em nível
  `DEBUG` (ver `./Security.md` §4).

---

## 6. Retenção

| Nível | Retenção padrão | Observação |
|-------|--------------------|--------------|
| `DEBUG` | Não retido em produção (desabilitado); em ambientes de teste, 7 dias. | |
| `INFO`/`WARN` | 30 dias | Alinhado à retenção operacional padrão do AIOS (`../024-Observability/`). |
| `ERROR`/`CRITICAL` | 180 dias | Retenção estendida para investigação de incidentes e auditoria de segurança. |

---

## 7. Exemplo de Linha de Log

```json
{
  "timestamp": "2026-07-21T12:07:11.442Z",
  "level": "CRITICAL",
  "message": "Syscall proibida bloqueada pelo perfil seccomp; sessão encerrada.",
  "component": "SandboxManager",
  "tenant_id": "acme",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "2b3c4d5e6f708192",
  "session_id": "01J9ZA1AGENTSESSN000000001",
  "runtime_id": "01J9ZA1RVNTMEXEC0000000001",
  "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "event": "sandbox.violation",
  "error_code": "AIOS-RUNTIME-0015",
  "violation_type": "egress_denied"
}
```

---

## 8. Referências

- Correlação e cabeçalhos: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6.
- Redação de PII (política): `./Security.md` §7.
- Catálogo de métricas correlatas: `./Metrics.md`.
- Stack de observabilidade (Serilog/Seq/OTel Collector): `../024-Observability/`.
