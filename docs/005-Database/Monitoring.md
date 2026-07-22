---
Documento: Monitoring
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0053, ADR-0055, ADR-0056
RFCs relacionados: RFC-0001
Depende de: Metrics.md, NonFunctionalRequirements.md, 024-Observability, 029-Operations
---

# 005-Database — Monitoramento

Dashboards e alertas do subsistema de dados. Todos os limiares abaixo derivam
diretamente dos SLO de `./NonFunctionalRequirements.md` — divergência é defeito.

---

## 1. Golden Signals do Módulo

| Sinal | Métrica principal | Meta |
|-------|-------------------|------|
| **Latência** | `aios_db_query_latency_ms` (p99 por `kind`) | 5 ms (pk) · 15 ms (write) · 50 ms (vector) |
| **Tráfego** | `aios_db_transactions_total` (taxa) | ≥ 20.000 tx/s sustentáveis |
| **Erros** | `aios_db_pool_rejections_total`, `aios_db_statement_timeouts_total` | tendência estável, sem picos sustentados |
| **Saturação** | `aios_db_pool_connections_waiting`, `aios_db_table_bloat_ratio` | fila ≈ 0 · bloat ≤ 20% |

Sinais adicionais, específicos deste módulo por serem obrigações e não desempenho:
**cobertura de RLS** (deve ser 1,0), **RPO observado** e **idade da última verificação
de backup**.

---

## 2. Dashboards

### 2.1 `DB / Visão Geral`
- Linha do tempo de p99 por `kind` de consulta (referência tracejada nas metas).
- Transações/s por `target` (primary × replica) — mostra o alívio das réplicas.
- Conexões ativas e em fila por serviço consumidor (bulkhead em ação).
- Erros por código (`AIOS-DB-*`) em barras empilhadas.

### 2.2 `DB / Durabilidade`
- `aios_db_rpo_seconds` com faixa verde (< 60 s), amarela (< 300 s) e vermelha.
- Último backup por tipo e idade da última verificação.
- Lag por réplica e elegibilidade de leitura.
- Histórico de restaurações com RTO medido.

### 2.3 `DB / Migrações`
- Migrações por estado terminal ao longo do tempo.
- Distribuição de `aios_db_migration_block_ms` com a linha de 200 ms.
- Top violações de lint por regra — mostra qual convenção mais tropeça e orienta
  documentação/tooling, em vez de virar atrito recorrente.

### 2.4 `DB / Conformidade`
- `aios_db_rls_coverage_ratio` (deve ser 1,0; qualquer outro valor é incidente).
- Violações por tipo, com a tabela responsável.
- Decisões do PDP por capability (`allow`/`deny`).
- Usos de `BYPASSRLS` com finalidade declarada.

### 2.5 `DB / Ciclo de vida do dado`
- Partições futuras por tabela (alerta se abaixo do mínimo).
- Linhas expurgadas por TTL e bytes recuperados.
- Pedidos de esquecimento por desfecho; tabelas sob `legal_hold`.

---

## 3. Regras de Alerta (Prometheus)

```yaml
groups:
- name: aios-database-critical
  rules:
  - alert: DbRlsCoverageBelowFull
    expr: aios_db_rls_coverage_ratio < 1
    for: 1m
    labels: { severity: P1, module: "005-Database" }
    annotations:
      summary: "Tabela multi-tenant sem RLS ativa"
      description: "NFR-011 violado. Ver evento database.compliance.violated."
      runbook: "../029-Operations/#db-rls-violation"

  - alert: DbRpoDegraded
    expr: aios_db_rpo_seconds > 300
    for: 2m
    labels: { severity: P1 }
    annotations:
      summary: "RPO acima de 5 min (NFR-009)"
      runbook: "../029-Operations/#db-wal-archive-stalled"

  - alert: DbPrimaryUnavailable
    expr: up{job="aios-postgres-primary"} == 0
    for: 30s
    labels: { severity: P1 }
    annotations:
      summary: "Primário indisponível; escritas rejeitadas (AIOS-DB-0005)"
      runbook: "../029-Operations/#db-failover"

  - alert: DbBackupNotVerified
    expr: aios_db_backup_verified_age_hours > 168
    for: 10m
    labels: { severity: P1 }
    annotations:
      summary: "Nenhum restore drill bem-sucedido na última semana (FR-012)"

  - alert: DbPartitionsExhausting
    expr: aios_db_partitions_ahead < 3
    for: 5m
    labels: { severity: P1 }
    annotations:
      summary: "Partições futuras insuficientes — risco de falha de escrita"
      runbook: "../029-Operations/#db-partition-precreate"

- name: aios-database-warning
  rules:
  - alert: DbQueryLatencyHigh
    expr: histogram_quantile(0.99, sum by (le) (rate(aios_db_query_latency_ms_bucket{kind="pk"}[10m]))) > 5
    for: 10m
    labels: { severity: P2 }
    annotations: { summary: "p99 de leitura por chave acima de 5 ms (NFR-001)" }

  - alert: DbVectorSearchSlow
    expr: histogram_quantile(0.99, sum by (le) (rate(aios_db_vector_search_latency_ms_bucket[15m]))) > 50
    for: 15m
    labels: { severity: P2 }
    annotations: { summary: "Busca vetorial acima de 50 ms (NFR-003)" }

  - alert: DbVectorRecallLow
    expr: aios_db_vector_recall_ratio < 0.95
    for: 30m
    labels: { severity: P2 }
    annotations: { summary: "Recall@10 abaixo de 0,95 (NFR-006) — avaliar ef_search" }

  - alert: DbReplicationLagHigh
    expr: histogram_quantile(0.95, sum by (le) (rate(aios_db_replication_lag_ms_bucket[10m]))) > 500
    for: 10m
    labels: { severity: P2 }
    annotations: { summary: "Lag p95 acima de 500 ms (NFR-013)" }

  - alert: DbPoolSaturated
    expr: aios_db_pool_connections_waiting > 0
    for: 5m
    labels: { severity: P2 }
    annotations: { summary: "Fila de conexões persistente — bulkhead saturado" }

  - alert: DbTableBloatHigh
    expr: aios_db_table_bloat_ratio > 0.2
    for: 1h
    labels: { severity: P3 }
    annotations: { summary: "Bloat acima de 20% (NFR-016) — revisar autovacuum" }

  - alert: DbOutboxBacklog
    expr: aios_db_outbox_pending > 10000
    for: 5m
    labels: { severity: P2 }
    annotations: { summary: "Relay do Outbox atrasado — eventos não publicados" }
```

---

## 4. SLO, SLI e Error Budget

| SLO | SLI | Janela | Budget |
|-----|-----|--------|--------|
| Disponibilidade ≥ 99,95% (NFR-005) | fração de minutos com escrita aceita | 30 dias | 21,9 min |
| p99 leitura por chave ≤ 5 ms (NFR-001) | fração de janelas de 5 min dentro da meta | 30 dias | 1% das janelas |
| RPO ≤ 5 min (NFR-009) | `aios_db_rpo_seconds` máximo | contínuo | **0 violações** |
| Cobertura de RLS = 100% (NFR-011) | `aios_db_rls_coverage_ratio` | contínuo | **0 violações** |

Ao esgotar o budget de disponibilidade, migrações de fase `contract` e mudanças de
configuração não críticas são **congeladas** até a recomposição.

---

## 5. Runbooks Associados

| Alerta | Runbook (`../029-Operations/`) | Primeira ação |
|--------|-------------------------------|---------------|
| `DbRlsCoverageBelowFull` | `#db-rls-violation` | Identificar a tabela pelo evento `compliance.violated`; migração emergencial reabilitando a política; auditar acessos recentes. |
| `DbRpoDegraded` | `#db-wal-archive-stalled` | Verificar conectividade com MinIO e espaço do buffer de WAL; se não resolver em minutos, escalar. |
| `DbPrimaryUnavailable` | `#db-failover` | Confirmar com `027-Cluster`; promover réplica de menor lag (UC-012). |
| `DbBackupNotVerified` | `#db-restore-drill` | Executar `POST /v1/database/backups/{urn}:verify` manualmente; se falhar, novo backup full imediato. |
| `DbPartitionsExhausting` | `#db-partition-precreate` | `POST /v1/database/partitions:ensure`; investigar por que o job não rodou. |
| `DbPoolSaturated` | `#db-pool-saturation` | Identificar o serviço com fila; avaliar aumento pontual de `max_connections_per_service` × correção do padrão de acesso. |
| `DbVectorRecallLow` | `#db-vector-tuning` | Aumentar `ef_search`; se insuficiente, planejar reindexação com `m` maior. |

---

## 6. Observação de Longo Prazo

Além dos alertas, o módulo mantém revisão periódica de tendências que não disparam
página, mas orientam capacidade:

- Crescimento de `aios_db_table_bytes` por schema (projeção de disco a 90 dias).
- Distribuição de `aios_db_tenant_storage_bytes` (identificação de *hot tenants*).
- Evolução de `aios_db_migration_block_ms` por módulo (quem está gerando DDL caro).
- Taxa de `aios_db_plan_regressions_total` após cada upgrade menor do PostgreSQL.

---

## 7. Referências

- Métricas: `./Metrics.md` · SLO: `./NonFunctionalRequirements.md`
- Logs: `./Logging.md` · Falhas: `./FailureRecovery.md`
- Runbooks: `../029-Operations/` · Observabilidade: `../024-Observability/`
