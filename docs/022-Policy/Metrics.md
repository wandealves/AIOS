---
Documento: Metrics
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0010, ADR-0008, ADR-0221
RFCs relacionados: RFC-0001
Depende de: NonFunctionalRequirements.md, Monitoring.md, 024-Observability
---

# 022-Policy — Catálogo de Métricas

Convenção: `aios_<subsistema>_<nome>_<unidade>`, subsistema **`pol`**. Exportadas via
OpenTelemetry → Prometheus (`../024-Observability/`).

## 1. Métricas de decisão (caminho quente)

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_pol_decision_latency_ms` | histogram | `path`, `surface`, `effect` | ms | Latência da avaliação, do `DecisionGateway` à resposta. `path ∈ {cached_attrs, remote_attrs}`; `surface ∈ {grpc, rest, nats}`. | NFR-001 |
| `aios_pol_decisions_total` | counter | `effect`, `reason_code`, `pep_module` | 1 | Decisões emitidas. Base da taxa de negação. | NFR-007 |
| `aios_pol_eval_timeout_total` | counter | `stage` | 1 | Avaliações que estouraram `pol.decision.eval_budget_ms`. | NFR-010 |
| `aios_pol_allow_without_rule_total` | counter | `pep_module` | 1 | `allow` sem regra casada. **DEVE ser sempre 0.** | NFR-008 |
| `aios_pol_rate_limited_total` | counter | `tenant` | 1 | Requisições recusadas por `AIOS-POL-0010`. | NFR-002 |
| `aios_pol_batch_size` | histogram | `surface` | 1 | Tamanho dos lotes em `EvaluateBatch`. | NFR-002 |
| `aios_pol_tenant_mismatch_total` | counter | `pep_module` | 1 | Tentativas com `tenant` divergente (`AIOS-POL-0003`). | NFR-016 |

> `aios_pol_allow_without_rule_total` é uma métrica que existe para **ser sempre
> zero**. Seu valor não informa desempenho: informa que o *default deny* foi rompido.
> Qualquer incremento é incidente de segurança, não degradação (`./Monitoring.md` §4).

## 2. Métricas de atributos

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_pol_attribute_resolve_ms` | histogram | `source_key`, `outcome` | ms | Latência de resolução por fonte. `outcome ∈ {cache_hit, resolved, timeout, error}`. |
| `aios_pol_attribute_unresolved_total` | counter | `source_key`, `criticality` | 1 | Atributos não resolvidos; com `criticality=required` implica `deny`. |
| `aios_pol_attribute_cache_hit_ratio` | gauge | `source_key` | ratio | Taxa de acerto do cache de atributo (0–1). |
| `aios_pol_attribute_source_status` | gauge | `source_key`, `status` | 1 | 1 para o `status` corrente (`active`/`degraded`/`disabled`). |

## 3. Métricas de bundle e propagação

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_pol_active_bundle_version` | gauge | `tenant`, `scope`, `replica` | 1 | Versão servida por cada réplica. Divergência = incidente. | NFR-004 |
| `aios_pol_bundle_propagation_ms` | histogram | `scope` | ms | Tempo entre `policy.bundle.updated` e adoção pela réplica. | NFR-004 |
| `aios_pol_bundle_transitions_total` | counter | `from`, `to` | 1 | Transições da FSM (`./StateMachine.md` §3). | — |
| `aios_pol_bundle_rules` | gauge | `tenant`, `scope` | 1 | Regras no bundle ativo. | NFR-005 |
| `aios_pol_bundle_compiled_bytes` | gauge | `tenant`, `scope` | bytes | Tamanho do artefato compilado em memória. | NFR-005 |
| `aios_pol_compile_duration_ms` | histogram | `outcome` | ms | Duração da compilação. `outcome ∈ {ok, rejected}`. | — |
| `aios_pol_conflicts_detected_total` | counter | `kind` | 1 | Conflitos na compilação. `kind ∈ {contradiction, shadowing, unreachable}`. | — |
| `aios_pol_signature_verification_failures_total` | counter | `stage` | 1 | Falhas de verificação de assinatura. `stage ∈ {activation, startup, reload}`. | NFR-015 |
| `aios_pol_no_active_bundle_total` | counter | `tenant` | 1 | Decisões negadas por ausência de bundle ativo (invariante I5). | — |

## 4. Métricas de governança

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_pol_test_coverage_ratio` | gauge | `tenant` | ratio | Cobertura do último bundle testado (0–1). | NFR-011 |
| `aios_pol_test_failures_total` | counter | `tenant`, `mandatory` | 1 | Casos golden reprovados. | NFR-011 |
| `aios_pol_simulation_duration_ms` | histogram | — | ms | Duração da simulação. | NFR-012 |
| `aios_pol_simulation_flipped_total` | counter | `direction` | 1 | Decisões que mudariam. `direction ∈ {to_deny, to_allow}`. | — |
| `aios_pol_exceptions_active` | gauge | `tenant` | 1 | Waivers vigentes. | — |
| `aios_pol_exception_grants_total` | counter | `tenant`, `outcome` | 1 | Waivers concedidos/negados. | — |
| `aios_pol_dual_approval_rejections_total` | counter | `operation` | 1 | Operações barradas por SoD (`AIOS-POL-0018`). | — |

## 5. Métricas de persistência e eventos

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_pol_journal_write_lag_ms` | histogram | — | ms | Atraso entre decidir e gravar no journal (escrita assíncrona). |
| `aios_pol_journal_dropped_total` | counter | `reason` | 1 | Registros de journal descartados (`reason ∈ {buffer_full, db_unavailable}`). **`deny` descartado é incidente.** |
| `aios_pol_outbox_pending` | gauge | — | 1 | Eventos no outbox aguardando publicação. |
| `aios_pol_outbox_publish_ms` | histogram | `subject` | ms | Latência de publicação do relay. |
| `aios_pol_events_consumed_total` | counter | `subject`, `outcome` | 1 | Eventos consumidos (`outcome ∈ {applied, duplicate, failed}`). |

## 6. Regras de cardinalidade

| Regra | Motivo |
|-------|--------|
| **NÃO DEVE** existir label `subject_urn`, `resource_urn` ou `decision_id`. | Cardinalidade ilimitada (10⁶ agentes) e exposição de identificadores em telemetria. |
| `tenant` é permitido apenas nas métricas de §3 e §4 (baixa frequência). | Nas métricas de decisão, `tenant` multiplicaria séries pelo número de tenants por segundo. |
| `reason_code` é permitido: conjunto **fechado** e estável (`./_DESIGN_BRIEF.md` §4.7). | Cardinalidade limitada por contrato. |
| Valores de atributo **NUNCA** viram label. | FR-023 (nenhum valor sensível em telemetria). |

Para análise por sujeito, use o `DecisionJournal` (`./Logging.md` §4) ou a trilha de
`../025-Audit/` — não a série temporal.

## 7. Métricas derivadas (recording rules)

```promql
# Taxa de negação por módulo PEP (base do alerta PolicyDenyRateSpike)
aios_pol_deny_ratio =
  sum by (pep_module) (rate(aios_pol_decisions_total{effect="deny"}[5m]))
/ sum by (pep_module) (rate(aios_pol_decisions_total[5m]))

# p99 de decisão no caminho com atributos em cache (SLI do NFR-001)
aios_pol_decision_latency_p99_cached =
  histogram_quantile(0.99,
    sum by (le) (rate(aios_pol_decision_latency_ms_bucket{path="cached_attrs"}[5m])))

# Convergência de versão de bundle entre réplicas (SLI do NFR-004)
aios_pol_bundle_version_skew =
  max by (tenant, scope) (aios_pol_active_bundle_version)
- min by (tenant, scope) (aios_pol_active_bundle_version)

# Vazão por réplica (SLI do NFR-002)
aios_pol_decisions_per_replica =
  sum by (replica) (rate(aios_pol_decisions_total[1m]))
```

## 8. Exemplo de exposição

```
# HELP aios_pol_decision_latency_ms Latência da avaliação de decisão
# TYPE aios_pol_decision_latency_ms histogram
aios_pol_decision_latency_ms_bucket{path="cached_attrs",surface="grpc",effect="allow",le="1"} 481203
aios_pol_decision_latency_ms_bucket{path="cached_attrs",surface="grpc",effect="allow",le="5"} 998110
aios_pol_decision_latency_ms_bucket{path="cached_attrs",surface="grpc",effect="allow",le="+Inf"} 998412

# HELP aios_pol_allow_without_rule_total Allow emitido sem regra casada (DEVE ser 0)
# TYPE aios_pol_allow_without_rule_total counter
aios_pol_allow_without_rule_total{pep_module="006-Kernel"} 0
```

## 9. Referências

- Metas: `./NonFunctionalRequirements.md` · Alertas: `./Monitoring.md`
- Logs correlacionados: `./Logging.md` · Benchmark: `./Benchmark.md`
- Plataforma de observabilidade: `../024-Observability/`
- Convenção de nomes: `../_templates/MODULE_TEMPLATE.md`

*Fim de `Metrics.md`.*
