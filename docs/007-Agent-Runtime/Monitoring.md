---
Documento: Monitoring
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0010 (global)
RFCs relacionados: RFC-0001
Depende de: ./Metrics.md, ./NonFunctionalRequirements.md, ../024-Observability/
---

# 007-Agent-Runtime — Monitoring

> Dashboards, alertas e *golden signals* do Agent Runtime, construídos sobre
> o catálogo de `./Metrics.md`. Metas de SLO/SLI são **idênticas** às de
> `./NonFunctionalRequirements.md` — nenhum limiar de alerta diverge das
> metas ali fixadas.

## Índice

1. Golden signals
2. Dashboards
3. Regras de alerta (Prometheus)
4. SLO/SLA/SLI — resumo operacional
5. Runbooks associados
6. Referências

---

## 1. Golden Signals

| Sinal | Métrica primária | Meta |
|-------|---------------------|------|
| Latência | `aios_runtime_cold_start_ms` (p99), `aios_runtime_step_latency_ms` (p95) | ≤ 250 ms (warm) / ≤ 900 ms (cold); ≤ 15 ms por passo (NFR-001, NFR-003). |
| Tráfego | `aios_runtime_events_published_total` (rate), `aios_runtime_sessions_active` | ≥ 5.000 msg/s por nó sem *backpressure* (NFR-007); ≥ 500 sessões/nó (NFR-004). |
| Erros | `aios_runtime_boot_failures_total`, `aios_runtime_sandbox_violations_total`, `aios_runtime_capability_denials_total` | 0 escapes de sandbox (NFR-008); taxa de boot failure sob controle operacional. |
| Saturação | `aios_runtime_session_rss_mib`, `aios_runtime_instances{state="busy"}` / `max_sessions_per_node` | ≤ 30 MiB overhead por sessão ociosa (NFR-010); densidade dentro do limite configurado. |

---

## 2. Dashboards

| Dashboard | Painéis principais | Público |
|-----------|------------------------|----------|
| **Runtime — Cold Start & Pool** | `aios_runtime_cold_start_ms` (p50/p95/p99, por `warm`), `aios_runtime_instances` por estado, taxa de `AIOS-RUNTIME-0001`. | SRE, Operações (`029-Operations`). |
| **Runtime — Loop Cognitivo** | `aios_runtime_step_latency_ms`, `aios_runtime_steps_total` por `phase`/`status`, `aios_runtime_replans_total`, `aios_runtime_watchdog_triggers_total`. | Equipe do módulo 007, Arquitetura. |
| **Runtime — Cotas e Custo** | `aios_runtime_quota_exceeded_total` por `resource`, `aios_runtime_cost_usd_total`, `aios_runtime_tokens_consumed_total`. | FinOps, Gestor de Custo (persona da Visão). |
| **Runtime — Dependências (Tool/Model)** | `aios_runtime_tool_circuit_breaker_state`, `aios_runtime_tool_invocation_latency_ms`, `aios_runtime_model_call_latency_ms`, `aios_runtime_model_fallback_total`. | SRE, Equipe do 015/017. |
| **Runtime — Segurança** | `aios_runtime_sandbox_violations_total`, `aios_runtime_capability_denials_total`, `aios_runtime_pii_redactions_total`. | CISO/Compliance, `025-Audit`. |
| **Runtime — Escala e Retomada** | `aios_runtime_resume_latency_ms`, `aios_runtime_checkpoint_integrity_failures_total`, `aios_runtime_lease_conflicts_total`. | SRE, Arquitetura. |

---

## 3. Regras de Alerta (Prometheus)

```yaml
groups:
  - name: agent-runtime.rules
    rules:
      - alert: RuntimeColdStartP99Breach
        expr: histogram_quantile(0.99, sum(rate(aios_runtime_cold_start_ms_bucket{warm="true"}[5m])) by (le, tenant)) > 250
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "Cold start p99 (pool quente) acima de 250ms para {{ $labels.tenant }}"
          runbook: "./FailureRecovery.md#31-falha-de-boot-do-sandbox"

      - alert: RuntimeStepLatencyP95Breach
        expr: histogram_quantile(0.95, sum(rate(aios_runtime_step_latency_ms_bucket[5m])) by (le, tenant)) > 15
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "Latência de passo p95 acima de 15ms para {{ $labels.tenant }}"

      - alert: RuntimeSandboxViolationDetected
        expr: increase(aios_runtime_sandbox_violations_total[5m]) > 0
        for: 0m
        labels: { severity: critical }
        annotations:
          summary: "Violação de sandbox detectada — {{ $labels.tenant }} / {{ $labels.violation_type }}"
          runbook: "./FailureRecovery.md#33-violação-de-sandbox"

      - alert: RuntimeCircuitBreakerOpen
        expr: aios_runtime_tool_circuit_breaker_state == 2
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "Circuit breaker de tool aberto há mais de 5min para {{ $labels.tenant }}"

      - alert: RuntimeOutboxBacklog
        expr: aios_runtime_events_outbox_pending > 1000
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "Outbox local com backlog acima de 1000 eventos pendentes"
          runbook: "./FailureRecovery.md#4-dlq-e-eventos-não-publicáveis"

      - alert: RuntimePoolSaturation
        expr: (aios_runtime_instances{state="busy"} / on(tenant) group_left aios_runtime_pool_max_sessions_per_node) > 0.9
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "Pool de runtime acima de 90% da capacidade configurada por nó"

      - alert: RuntimeAvailabilityBudgetBurn
        expr: aios_runtime_availability_ratio < 0.999
        for: 15m
        labels: { severity: critical }
        annotations:
          summary: "Disponibilidade do pool abaixo de 99,9% (NFR-005) para {{ $labels.tenant }}"

      - alert: RuntimeResumeRtoBreach
        expr: histogram_quantile(0.99, sum(rate(aios_runtime_resume_latency_ms_bucket[5m])) by (le, tenant)) > 30000
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "Retomada de sessão (resume) acima do RTO de 30s (NFR-006)"
```

---

## 4. SLO/SLA/SLI — Resumo Operacional

| SLO | SLI | Janela | Error Budget |
|-----|-----|--------|-----------------|
| Cold start p99 ≤ 250 ms (warm) | `aios_runtime_cold_start_ms` | 30 dias móveis | Consumo de budget monitorado por painel dedicado; 3 violações consecutivas de 10 min disparam revisão de capacidade do pool quente. |
| Disponibilidade do pool ≥ 99,9% | `aios_runtime_availability_ratio` | 30 dias móveis | ~43 min/mês de indisponibilidade tolerada; consumo acelerado dispara *freeze* de mudanças não emergenciais. |
| 0 escapes de sandbox | `aios_runtime_sandbox_violations_total` (com efeito fora do sandbox) | Contínuo | Não há orçamento — qualquer escape confirmado é incidente de severidade máxima (ver `./Security.md`). |
| RTO de retomada ≤ 30 s | `aios_runtime_resume_latency_ms` | 30 dias móveis | Violação recorrente dispara revisão de `checkpoint.interval_steps`/capacidade de MinIO. |

---

## 5. Runbooks Associados

| Cenário | Runbook |
|---------|------------|
| Cold start degradado sob pico | `./FailureRecovery.md` — seção "Falha de boot do sandbox" e ajuste de `runtime.pool.warm_size`. |
| Violação de sandbox | `./FailureRecovery.md` — seção "Violação de sandbox"; escalonamento imediato a `025-Audit`/`021-Security`. |
| Backlog de outbox | `./FailureRecovery.md` — seção "Perda de conexão NATS". |
| Circuit breaker preso em `open` | `./FailureRecovery.md` — seção "Timeout/erro de tool (015)". |
| Saturação de pool | `./Scalability.md` — seção "Caminho de escala"; acionar autoscaling do Runtime Supervisor. |

---

## 6. Referências

- Catálogo de métricas: `./Metrics.md`.
- Metas de SLO (fonte): `./NonFunctionalRequirements.md`.
- Modos de falha e recuperação: `./FailureRecovery.md`.
- Stack de observabilidade (Prometheus/Grafana/OTel Collector): `../024-Observability/`.
