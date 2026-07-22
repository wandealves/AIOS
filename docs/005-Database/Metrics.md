---
Documento: Metrics
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0053, ADR-0055, ADR-0056, ADR-0057
RFCs relacionados: RFC-0001
Depende de: 024-Observability, NonFunctionalRequirements.md, Monitoring.md
---

# 005-Database — Catálogo de Métricas

Convenção de nome: `aios_<subsistema>_<nome>_<unidade>`, subsistema = **`db`**.
Exportação **OpenTelemetry → Prometheus** (`../024-Observability/`). Labels comuns a
todas as métricas: `service`, `instance`, `env`. Labels de alta cardinalidade
(`tenant_id`, `urn`) **NÃO DEVEM** ser usados como label de métrica — a correlação por
tenant é feita via traces e logs (`./Logging.md`).

---

## 1. Desempenho de Consulta

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_db_query_latency_ms` | histogram | `kind` (pk\|scan\|write\|vector), `schema` | ms | Latência de consulta observada no pool. | NFR-001, NFR-002 |
| `aios_db_transactions_total` | counter | `result` (commit\|rollback), `target` (primary\|replica) | 1 | Transações concluídas. | NFR-004 |
| `aios_db_slow_queries_total` | counter | `schema`, `reason` (threshold\|timeout) | 1 | Consultas acima de `db.query.slow_threshold_ms` ou abortadas. | NFR-004 |
| `aios_db_deadlocks_total` | counter | `schema` | 1 | Deadlocks detectados pelo PostgreSQL. | NFR-012 |
| `aios_db_plan_regressions_total` | counter | `schema` | 1 | Regressões de plano identificadas pelo `QueryGovernor`. | NFR-004 |

Buckets recomendados para `aios_db_query_latency_ms`:
`[0.5, 1, 2, 5, 10, 15, 25, 50, 100, 250, 500, 1000, 5000]` — cobre a meta de 5 ms
(PK), 15 ms (escrita) e 50 ms (vetorial) com resolução útil na cauda.

---

## 2. Conexões e Contenção

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_db_pool_connections_active` | gauge | `service`, `target` | 1 | Conexões ativas por serviço consumidor. | NFR-004 |
| `aios_db_pool_connections_waiting` | gauge | `service` | 1 | Clientes na fila do PgBouncer (sinal de bulkhead saturado). | NFR-004 |
| `aios_db_pool_rejections_total` | counter | `service`, `code` | 1 | Rejeições `AIOS-DB-0006`. | FR-015 |
| `aios_db_statement_timeouts_total` | counter | `schema` | 1 | Abortos por `statement_timeout` (`AIOS-DB-0007`). | FR-015 |

---

## 3. Migração

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_db_migration_duration_ms` | histogram | `phase`, `owner_module` | ms | Duração de aplicação. | NFR-010 |
| `aios_db_migration_block_ms` | histogram | `phase`, `owner_module` | ms | Tempo de bloqueio de escrita causado pela migração. | NFR-010 |
| `aios_db_migrations_total` | counter | `state` (Applied\|Failed\|RolledBack), `phase` | 1 | Migrações por estado terminal. | FR-003 |
| `aios_db_migration_lock_wait_ms` | histogram | — | ms | Espera pelo advisory lock. | FR-003 |
| `aios_db_migration_lint_violations_total` | counter | `rule` | 1 | Violações por regra de convenção. | FR-002 |

Buckets de `aios_db_migration_block_ms`: `[5, 10, 25, 50, 100, 200, 500, 1000, 5000]` —
o bucket **200 ms** é o limite de NFR-010 e é o alvo do alerta.

---

## 4. Partição, Retenção e Armazenamento

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_db_partitions_ahead` | gauge | `schema`, `table` | 1 | Partições futuras existentes (alerta se < `precreate_ahead`). | FR-007 |
| `aios_db_partitions_dropped_total` | counter | `schema`, `table` | 1 | Partições removidas por retenção. | FR-008 |
| `aios_db_retention_rows_removed_total` | counter | `schema`, `table`, `method` | 1 | Linhas expurgadas por TTL. | FR-008 |
| `aios_db_retention_cycle_duration_ms` | histogram | — | ms | Duração do ciclo do `RetentionEnforcer`. | FR-008 |
| `aios_db_table_bytes` | gauge | `schema`, `table` | bytes | Tamanho em disco (insumo de custo). | FR-020 |
| `aios_db_table_bloat_ratio` | gauge | `schema`, `table` | ratio | Proporção estimada de espaço morto. | NFR-016 |
| `aios_db_tenant_storage_bytes` | gauge | `bucket` (faixa de uso) | bytes | Distribuição de consumo por tenant, sem cardinalidade por tenant. | FR-020 |

---

## 5. Durabilidade e Replicação

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_db_replication_lag_ms` | histogram | `replica` | ms | Lag de aplicação de WAL por réplica. | NFR-013 |
| `aios_db_replica_read_eligible` | gauge | `replica` | 0/1 | Réplica elegível para leitura. | FR-016 |
| `aios_db_rpo_seconds` | gauge | — | s | Segundos desde o último WAL arquivado com sucesso. | NFR-009 |
| `aios_db_backup_last_success_timestamp` | gauge | `kind` | unix s | Momento do último backup bem-sucedido. | FR-011 |
| `aios_db_backup_verified_age_hours` | gauge | — | h | Idade da última verificação de restauração. | FR-012 |
| `aios_db_restore_duration_ms` | histogram | `kind` (drill\|incident) | ms | Duração de restauração (base do RTO). | NFR-009 |
| `aios_db_wal_archive_failures_total` | counter | — | 1 | Falhas de arquivamento de WAL. | NFR-009 |

---

## 6. Conformidade e Segurança

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_db_rls_coverage_ratio` | gauge | — | ratio | Tabelas multi-tenant com RLS ativa ÷ total. Meta: **1,0**. | NFR-011 |
| `aios_db_compliance_violations_total` | counter | `violation` (RLS_DISABLED\|RETENTION_MISSING\|LEGAL_BASIS_MISSING) | 1 | Violações detectadas pela auditoria. | NFR-011 |
| `aios_db_policy_decisions_total` | counter | `capability`, `decision` (allow\|deny) | 1 | Decisões do PDP para operações administrativas. | FR-017 |
| `aios_db_bypassrls_uses_total` | counter | `purpose` | 1 | Usos da *role* `platform_admin`. | `./Security.md` §3 |
| `aios_db_erasure_requests_total` | counter | `status` | 1 | Pedidos de esquecimento por desfecho. | FR-009 |

---

## 7. Extensões

| Métrica | Tipo | Labels | Unidade | Semântica | NFR |
|---------|------|--------|---------|-----------|-----|
| `aios_db_vector_search_latency_ms` | histogram | `schema`, `index_type` | ms | Latência de busca top-K. | NFR-003 |
| `aios_db_vector_recall_ratio` | gauge | `schema`, `table`, `column` | ratio | Recall@10 medido contra busca exata. Meta ≥ **0,95**. | NFR-006 |
| `aios_db_vector_index_bytes` | gauge | `schema`, `table`, `column` | bytes | Tamanho do índice vetorial. | NFR-016 |
| `aios_db_graph_traversal_depth` | histogram | `tenant_bucket` | 1 | Profundidade das travessias AGE executadas. | FR-014 |
| `aios_db_graph_rejected_total` | counter | `reason` (depth\|timeout) | 1 | Travessias rejeitadas por limite. | `./Security.md` §5 |

---

## 8. Operação do Serviço

| Métrica | Tipo | Labels | Unidade | Semântica |
|---------|------|--------|---------|-----------|
| `aios_db_outbox_pending` | gauge | `schema` | 1 | Eventos pendentes de publicação (sinal de atraso do relay). |
| `aios_db_outbox_publish_latency_ms` | histogram | — | ms | Atraso entre `COMMIT` e publicação no NATS. |
| `aios_db_config_pending_reload` | gauge | — | 1 | Chaves não recarregáveis alteradas aguardando reinício. |
| `aios_db_jobs_last_run_timestamp` | gauge | `job` (retention\|partition\|backup\|audit) | unix s | Última execução bem-sucedida de cada job. |

---

## 9. Exemplos de Consulta

```promql
# Cobertura de RLS deve ser exatamente 1
aios_db_rls_coverage_ratio < 1

# p99 de busca vetorial (NFR-003)
histogram_quantile(0.99, sum by (le) (rate(aios_db_vector_search_latency_ms_bucket[15m]))) > 50

# Partições futuras insuficientes (risco de falha de escrita ao virar o período)
aios_db_partitions_ahead < 3

# RPO degradado (NFR-009)
aios_db_rpo_seconds > 300

# Backup sem verificação recente (FR-012)
aios_db_backup_verified_age_hours > 168
```

---

## 10. Referências

- NFR e metas: `./NonFunctionalRequirements.md`
- Alertas e dashboards: `./Monitoring.md` · Logs: `./Logging.md`
- Plataforma de observabilidade: `../024-Observability/`
