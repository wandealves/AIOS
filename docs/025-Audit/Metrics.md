---
Documento: Metrics
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0010, ADR-0251, ADR-0255, ADR-0256
RFCs relacionados: RFC-0001, RFC-0240
Depende de: NonFunctionalRequirements.md, Monitoring.md, 024-Observability
---

# 025-Audit — Catálogo de Métricas

Convenção: `aios_<subsistema>_<nome>_<unidade>`, subsistema **`aud`**. Exportadas via
OpenTelemetry → Prometheus e **registradas** como `SignalDescriptor` no
`../024-Observability/` (RFC-0240), com `owner_module = 025-Audit`.

> **Métricas deste módulo descrevem o pipeline, nunca o conteúdo auditado.** Uma
> métrica com `subject_ref` ou `resource_urn` como label vazaria, por telemetria de
> baixa proteção, exatamente o que está cifrado dentro da trilha (§6).

## 1. Métricas de ingestão e durabilidade

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_aud_append_latency_ms` | histogram | `source`, `outcome` | ms | Latência de `AppendRecord` até o commit durável. `source ∈ {api, event}`. Buckets: 5, 10, 20, 50, 200. | NFR-001 |
| `aios_aud_commit_latency_ms` | histogram | — | ms | Parcela da latência gasta no commit síncrono (replicação). | NFR-001 |
| `aios_aud_records_total` | counter | `record_class`, `outcome`, `producer_module` | 1 | Registros gravados. | NFR-002 |
| `aios_aud_duplicates_total` | counter | `source` | 1 | Reingestões idempotentes (esperado > 0 com JetStream). | NFR-014 |
| `aios_aud_durable_ack_failed_total` | counter | `reason` | 1 | Escritas recusadas com `AIOS-AUD-0005`. **Não é defeito** — é o desenho funcionando. | NFR-006 |
| `aios_aud_ingest_rejected_total` | counter | `error_code` | 1 | Rejeições (`unregistered_class`, `bad_envelope`, `tenant_mismatch`). | — |
| `aios_aud_consumer_lag_s` | gauge | `stream` | s | Atraso do consumidor JetStream. | NFR-005 |

> `aios_aud_durable_ack_failed_total` merece leitura cuidadosa: cada incremento é um
> `503` devolvido a um produtor — e é **preferível** ao alternativo, que seria um
> registro aceito e perdido. O que se monitora é a **taxa** e sua persistência, não a
> existência.

## 2. Métricas de cadeia e selagem

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_aud_chain_length` | gauge | `partition_key` | 1 | Último `seq` da partição. | NFR-009 |
| `aios_aud_unsealed_records` | gauge | `partition_key` | 1 | Registros em `Received` aguardando selo. | NFR-004 |
| `aios_aud_seal_lag_s` | histogram | `partition_key` | s | Tempo entre gravação e selagem. Buckets: 10, 30, 60, 120, 600. | NFR-004 |
| `aios_aud_seals_total` | counter | `outcome` | 1 | Selos produzidos. `outcome ∈ {ok, signing_failed}`. | — |
| `aios_aud_seal_records` | histogram | — | 1 | Registros por selo. | — |
| `aios_aud_signing_latency_ms` | histogram | — | ms | Latência da assinatura no `../021-Security/`. | NFR-004 |

## 3. Métricas de integridade e completude

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_aud_chain_break_total` | counter | `partition_key` | 1 | Quebras de cadeia detectadas. **DEVE ser sempre 0.** | NFR-007 |
| `aios_aud_sequence_gap_total` | counter | `partition_key` | 1 | Lacunas de sequência detectadas. **DEVE ser sempre 0.** | NFR-005 |
| `aios_aud_verify_records_total` | counter | `mode`, `result` | 1 | Registros verificados. `mode ∈ {sample, full}`; `result ∈ {ok, mismatch}`. | NFR-007 |
| `aios_aud_verify_duration_ms` | histogram | `mode` | ms | Duração do ciclo de verificação. | NFR-008 |
| `aios_aud_verify_last_success_timestamp` | gauge | `mode` | s | *Timestamp* Unix da última verificação bem-sucedida. | NFR-007 |
| `aios_aud_worm_mismatch_total` | counter | — | 1 | Divergências entre banco e cópia WORM. **DEVE ser sempre 0.** | NFR-007 |
| `aios_aud_class_silent_total` | counter | `record_class`, `producer_module` | 1 | Classes com volume abaixo do esperado. | NFR-005 |
| `aios_aud_class_volume_ratio` | gauge | `record_class` | ratio | Volume observado / `expected_rate_per_day`. | NFR-005 |

> Três métricas deste módulo existem para **ser sempre zero**:
> `aios_aud_chain_break_total`, `aios_aud_sequence_gap_total` e
> `aios_aud_worm_mismatch_total`. Seus valores não informam desempenho: informam que
> uma garantia foi rompida. Qualquer incremento é incidente, não degradação.

## 4. Métricas de arquivamento e retenção

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_aud_archive_lag_s` | histogram | — | s | Tempo entre selagem e arquivamento WORM. | NFR-007 |
| `aios_aud_archived_seals_total` | counter | `outcome` | 1 | Selos arquivados. | — |
| `aios_aud_retention_purged_total` | counter | `record_class` | 1 | Payloads expurgados por fim de retenção. | NFR-010 |
| `aios_aud_retention_blocked_total` | counter | `reason` | 1 | Expurgos bloqueados. `reason = legal_hold` é o esperado. | NFR-010 |
| `aios_aud_payload_bytes` | gauge | `record_class` | bytes | Volume de payload ativo por classe. | NFR-009 |

## 5. Métricas de governança, consulta e privacidade

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_aud_holds_active` | gauge | `has_expiry` | 1 | *Holds* vigentes. `has_expiry ∈ {true,false}` — o segundo alimenta a revisão. | NFR-010 |
| `aios_aud_hold_age_days` | histogram | — | d | Idade dos *holds* ativos. | — |
| `aios_aud_erasures_total` | counter | `kind`, `outcome` | 1 | Apagamentos executados. | NFR-011 |
| `aios_aud_erasure_latency_h` | histogram | — | h | Tempo entre pedido e efetivação. | NFR-011 |
| `aios_aud_erasure_blocked_total` | counter | `reason` | 1 | Apagamentos bloqueados por *hold*. | NFR-010 |
| `aios_aud_query_latency_ms` | histogram | `by`, `outcome` | ms | Latência de consulta. `by ∈ {subject, decision, trace, class, window}`. | NFR-012 |
| `aios_aud_query_records_returned` | histogram | — | 1 | Registros por consulta. | NFR-012 |
| `aios_aud_query_denied_total` | counter | `capability` | 1 | Consultas negadas pelo PDP. | NFR-015 |
| `aios_aud_access_records_total` | counter | `principal_kind` | 1 | Registros de acesso gerados pela própria consulta (ADR-0257). | — |
| `aios_aud_export_duration_ms` | histogram | `outcome` | ms | Duração da montagem do pacote. | NFR-013 |
| `aios_aud_exports_total` | counter | `outcome` | 1 | Exportações. | — |

## 6. Regras de cardinalidade

| Regra | Motivo |
|-------|--------|
| **NÃO DEVE** existir label `subject_ref`, `resource_urn`, `record_hash`, `id` ou `decision_ref`. | Cardinalidade ilimitada **e** vazamento: esses valores identificam o fato auditado. |
| `partition_key` é permitido. | Cardinalidade = `aud.chain.partitions_per_tenant` × tenants — limitada e conhecida. |
| `record_class` e `producer_module` são permitidos. | Conjuntos **fechados**, registrados em `audit.auditable_class`. |
| `principal_kind` (não `principal_urn`) em métricas de acesso. | Quem consultou vive na **trilha**, não na métrica. |
| Buckets explícitos alinhados aos SLO (20 ms, 60 s, 24 h). | Buckets automáticos não caem nas fronteiras; o quantil seria interpolado errado. |

> A regra mais importante: **quem consultou a trilha aparece na trilha, não na
> métrica**. A métrica diz *quantas* consultas houve; o registro de acesso diz *quem*,
> *o quê* e *com que escopo* — sob PEP e cifrado.

## 7. Métricas derivadas (recording rules)

```promql
# p99 de escrita durável — SLI do NFR-001
aios_aud_append_latency_p99 =
  histogram_quantile(0.99, sum by (le) (rate(aios_aud_append_latency_ms_bucket[5m])))

# Fração da latência gasta no commit síncrono (diagnóstico de NFR-001)
aios_aud_commit_share =
  histogram_quantile(0.99, sum by (le) (rate(aios_aud_commit_latency_ms_bucket[5m])))
/ aios_aud_append_latency_p99

# Registros aguardando selo — SLI do NFR-004
aios_aud_unsealed_total = sum by (tenant) (aios_aud_unsealed_records)

# Idade da última verificação bem-sucedida — SLI do NFR-007
aios_aud_verify_staleness_s =
  time() - max by (mode) (aios_aud_verify_last_success_timestamp)

# Taxa de recusa por durabilidade (esperada ~0 fora de incidente)
aios_aud_durable_reject_ratio =
  sum(rate(aios_aud_durable_ack_failed_total[5m]))
/ sum(rate(aios_aud_records_total[5m]) + rate(aios_aud_durable_ack_failed_total[5m]))
```

## 8. Exemplo de exposição

```
# HELP aios_aud_chain_break_total Quebras de cadeia detectadas (DEVE ser 0)
# TYPE aios_aud_chain_break_total counter
aios_aud_chain_break_total{partition_key="acme:07"} 0

# HELP aios_aud_append_latency_ms Latência de AppendRecord até o commit durável
# TYPE aios_aud_append_latency_ms histogram
aios_aud_append_latency_ms_bucket{source="event",outcome="ok",le="20"} 4812033
aios_aud_append_latency_ms_bucket{source="event",outcome="ok",le="+Inf"} 4813991

# HELP aios_aud_holds_active Legal holds vigentes
# TYPE aios_aud_holds_active gauge
aios_aud_holds_active{has_expiry="false"} 3
aios_aud_holds_active{has_expiry="true"} 11
```

## 9. Referências

- Metas: `./NonFunctionalRequirements.md` · Alertas: `./Monitoring.md`
- Logs correlacionados: `./Logging.md` · Benchmark: `./Benchmark.md`
- Convenção normativa de sinais: `../024-Observability/` (RFC-0240)
- Convenção de nomes: `../_templates/MODULE_TEMPLATE.md`

*Fim de `Metrics.md`.*
