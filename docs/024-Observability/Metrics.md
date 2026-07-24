---
Documento: Metrics
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0010, ADR-0241, ADR-0242, ADR-0243
RFCs relacionados: RFC-0001, RFC-0240
Depende de: NonFunctionalRequirements.md, Monitoring.md, _DESIGN_BRIEF.md §3.1
---

# 024-Observability — Catálogo de Métricas

Convenção: `aios_<subsistema>_<nome>_<unidade>`, subsistema **`obs`**. Este módulo é
o **guardião** da convenção (RFC-0240) e, portanto, o primeiro a cumpri-la: todas as
métricas abaixo estão registradas como `SignalDescriptor` com `owner_module = 024-Observability`.

## 1. Métricas de ingestão

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_obs_ingest_latency_ms` | histogram | `signal_kind`, `transport` | ms | Latência da aceitação do lote OTLP (até o enfileiramento). Buckets: 5, 10, 25, 50, 100, 500. | NFR-001 |
| `aios_obs_received_total` | counter | `signal_kind`, `tenant` | 1 | Datapoints/spans/logs recebidos. | NFR-006 |
| `aios_obs_dropped_total` | counter | `signal_kind`, `reason` | 1 | **Descartados**. `reason ∈ {queue_full, quarantined, unregistered, storage_down, batch_too_large}`. | NFR-006 |
| `aios_obs_queue_depth` | gauge | `pipeline` | 1 | Profundidade da fila interna. | NFR-003 |
| `aios_obs_batch_bytes` | histogram | `transport` | bytes | Tamanho dos lotes recebidos. | NFR-003 |
| `aios_obs_emitter_rejected_total` | counter | `reason` | 1 | Lotes rejeitados na borda (`tenant_mismatch`, `bad_mtls`). | NFR-017 |

> `aios_obs_dropped_total` é a métrica mais importante deste módulo. O *fail-open*
> (ADR-0243) torna a perda **silenciosa por construção**; esta série é o que a torna
> audível. Um sistema em que ela não é monitorada tomou a decisão de perder telemetria
> sem a decisão de saber quanto.

## 2. Métricas de admissão e cardinalidade

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_obs_active_series` | gauge | `tenant`, `signal_name` | 1 | Séries ativas. Base do orçamento. | NFR-009 |
| `aios_obs_cardinality_budget_ratio` | gauge | `tenant`, `scope` | ratio | `current_series / max_active_series` (0–1). | NFR-009 |
| `aios_obs_signal_quarantined_total` | counter | `reason`, `owner_module` | 1 | Quarentenas. `reason ∈ {unregistered, cardinality, convention}`. | — |
| `aios_obs_signal_registered_total` | counter | `signal_kind`, `owner_module` | 1 | Registros de sinal. | — |
| `aios_obs_labels_dropped_total` | counter | `signal_name` | 1 | Labels removidos por não constarem em `allowed_labels`. | NFR-009 |
| `aios_obs_forbidden_label_total` | counter | `label`, `owner_module` | 1 | Tentativas de usar label de identidade. **Deveria tender a 0.** | NFR-009 |

## 3. Métricas de privacidade e amostragem

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_obs_pii_redacted_total` | counter | `attribute_kind`, `mode` | 1 | Atributos redigidos na borda. | NFR-014 |
| `aios_obs_pii_undeclared_total` | counter | `owner_module` | 1 | PII detectada em sinal **não** declarado como `pii` — defeito do descritor. | NFR-014 |
| `aios_obs_trace_sampled_total` | counter | `decision`, `reason` | 1 | Decisões de amostragem. `reason ∈ {error, slow, ratio}`. | — |
| `aios_obs_trace_fragmented_total` | counter | — | 1 | Traces montados incompletos (spans em gateways distintos). | NFR-003 |

## 4. Métricas de SLO e alerta

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_obs_slo_sli_ratio` | gauge | `slo_key`, `owner_module` | ratio | SLI corrente (0–1). | NFR-008 |
| `aios_obs_slo_budget_consumed_ratio` | gauge | `slo_key` | ratio | Fração do error budget consumida. | — |
| `aios_obs_slo_burn_rate` | gauge | `slo_key`, `window` | 1 | Taxa de queima. `window ∈ {1h, 6h, 24h}`. | NFR-008 |
| `aios_obs_alert_transitions_total` | counter | `from`, `to`, `severity` | 1 | Transições da FSM (`./StateMachine.md` §3). | — |
| `aios_obs_alert_open` | gauge | `severity`, `state` | 1 | Instâncias abertas por estado. | NFR-005 |
| `aios_obs_alert_notify_latency_ms` | histogram | `severity`, `channel` | ms | Tempo entre `Firing` e notificação entregue. | NFR-005 |
| `aios_obs_alert_flapping_total` | counter | `rule_key` | 1 | Alternâncias `Firing`↔`Resolved` na janela. | — |
| `aios_obs_alert_nodata_total` | counter | `rule_key` | 1 | Instâncias que foram a `Expired` por ausência de dado. | — |
| `aios_obs_silence_active` | gauge | `tenant` | 1 | Silêncios vigentes. | — |
| `aios_obs_silence_created_total` | counter | `rule_key` | 1 | Silêncios criados, por regra alvo. Base do alerta de reincidência. | — |
| `aios_obs_runbook_stale_total` | gauge | — | 1 | Runbooks sem revisão há mais de 180 dias. | NFR-016 |

## 5. Métricas de consulta, retenção e eventos

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_obs_query_latency_ms` | histogram | `store`, `outcome` | ms | Latência de consulta. `store ∈ {metrics, traces, logs}`. | NFR-013 |
| `aios_obs_query_series_scanned` | histogram | `store` | 1 | Séries varridas por consulta. | NFR-013 |
| `aios_obs_query_aborted_total` | counter | `reason` | 1 | Consultas abortadas (`timeout`, `series_budget`). | NFR-013 |
| `aios_obs_downsample_duration_ms` | histogram | `from_tier`, `to_tier` | ms | Duração do *downsampling*. | NFR-012 |
| `aios_obs_downsample_last_success_timestamp` | gauge | `to_tier` | s | *Timestamp* Unix da última execução bem-sucedida. Base do alerta de atraso. | NFR-012 |
| `aios_obs_retention_purged_total` | counter | `tier`, `signal_kind` | 1 | Registros expurgados. | NFR-012 |
| `aios_obs_erasure_receipts_total` | counter | `outcome` | 1 | Comprovantes de direito ao esquecimento. | NFR-014 |
| `aios_obs_outbox_pending` | gauge | — | 1 | Eventos aguardando publicação. | — |
| `aios_obs_events_consumed_total` | counter | `subject`, `outcome` | 1 | Eventos consumidos (`applied`, `duplicate`, `failed`). | — |

## 6. Regras de cardinalidade (aplicadas a si mesmo)

| Regra | Motivo |
|-------|--------|
| **NÃO DEVE** existir label `agent_urn`, `task_urn`, `trace_id` ou `decision_id`. | Cardinalidade ilimitada; a granularidade individual vive em trace/log. |
| `signal_name` é permitido em `aios_obs_active_series` e `aios_obs_labels_dropped_total`. | Conjunto limitado pelo número de sinais **registrados** — que é exatamente o que o módulo controla. |
| `tenant` é permitido nas métricas de cardinalidade, ingestão e silêncio. | Necessário para orçamento por tenant; cardinalidade = nº de tenants, controlada. |
| `reason` e `outcome` são conjuntos **fechados**. | Documentados nesta tabela; valor novo exige nova versão de descritor. |
| Buckets de histograma **DEVEM** ser explícitos e alinhados aos limiares de SLO. | Buckets automáticos não caem nas fronteiras; `histogram_quantile` sobre eles devolve um número que **parece** um p99. |

> Este módulo é o primeiro a cumprir a regra que impõe. Um catálogo de observabilidade
> que usasse `trace_id` como label estaria demonstrando, no próprio corpo, o defeito
> que o `CardinalityGuard` existe para impedir.

## 7. Métricas derivadas (recording rules)

```promql
# Taxa de descarte — SLI do NFR-006
aios_obs_drop_ratio =
  sum by (signal_kind) (rate(aios_obs_dropped_total[5m]))
/ sum by (signal_kind) (rate(aios_obs_received_total[5m]))

# p99 de ingestão — SLI do NFR-001
aios_obs_ingest_latency_p99 =
  histogram_quantile(0.99, sum by (le) (rate(aios_obs_ingest_latency_ms_bucket[5m])))

# Ocupação do orçamento de cardinalidade — base do alerta preventivo
aios_obs_cardinality_pressure =
  max by (tenant) (aios_obs_cardinality_budget_ratio)

# Latência de notificação de alertas críticos — SLI do NFR-005
aios_obs_notify_latency_p95_critical =
  histogram_quantile(0.95,
    sum by (le) (rate(aios_obs_alert_notify_latency_ms_bucket{severity="critical"}[5m])))

# Traces retidos por motivo — verifica a política tail-based (ADR-0244)
aios_obs_trace_keep_ratio =
  sum by (reason) (rate(aios_obs_trace_sampled_total{decision="keep"}[5m]))
/ sum       (rate(aios_obs_trace_sampled_total[5m]))
```

## 8. Exemplo de exposição

```
# HELP aios_obs_dropped_total Telemetria descartada (fail-open, ADR-0243)
# TYPE aios_obs_dropped_total counter
aios_obs_dropped_total{signal_kind="metric",reason="queue_full"} 41200
aios_obs_dropped_total{signal_kind="span",reason="quarantined"} 118

# HELP aios_obs_forbidden_label_total Tentativas de usar label de identidade
# TYPE aios_obs_forbidden_label_total counter
aios_obs_forbidden_label_total{label="agent_urn",owner_module="006-Kernel"} 0

# HELP aios_obs_ingest_latency_ms Latência de aceitação do lote OTLP
# TYPE aios_obs_ingest_latency_ms histogram
aios_obs_ingest_latency_ms_bucket{signal_kind="metric",transport="grpc",le="50"} 984210
aios_obs_ingest_latency_ms_bucket{signal_kind="metric",transport="grpc",le="+Inf"} 984733
```

## 9. Referências

- Metas: `./NonFunctionalRequirements.md` · Alertas: `./Monitoring.md`
- Logs correlacionados: `./Logging.md` · Benchmark: `./Benchmark.md`
- Convenção normativa: `./_DESIGN_BRIEF.md` §3.1 e RFC-0240 (`./RFC.md`)
- Convenção de nomes: `../_templates/MODULE_TEMPLATE.md`

*Fim de `Metrics.md`.*
