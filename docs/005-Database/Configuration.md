---
Documento: Configuration
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0053, ADR-0054, ADR-0055, ADR-0056, ADR-0057, ADR-0058, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050
Depende de: _DESIGN_BRIEF.md §8, Database.md, Deployment.md, 021-Security (segredos)
---

# 005-Database — Configuração

Prefixo de variável de ambiente: **`AIOS_DB_`**. A chave `db.pool.mode` corresponde à
variável `AIOS_DB_POOL_MODE`, e assim por diante (pontos → `_`, maiúsculas).

Escopos: `global` (cluster), `tenant`, `table`. **Recarregável** indica *hot reload*
sem reinício do serviço; chaves não recarregáveis exigem reinício ou migração.

Os valores abaixo são idênticos aos de `./_DESIGN_BRIEF.md` §8 e coerentes com
`./Database.md` §11 — divergência é defeito.

---

## 1. Catálogo de Chaves

### 1.1 Acesso e concorrência

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `db.pool.max_connections_per_service` | int | 50 | 1–500 | global | sim | Bulkhead de conexões por serviço no PgBouncer (`AIOS-DB-0006` ao exceder). |
| `db.pool.mode` | enum | `transaction` | session\|transaction\|statement | global | **não** | Modo do PgBouncer. `transaction` é o default do AIOS (ADR-0059). |
| `db.query.statement_timeout_ms` | int | 30000 | 100–600000 | global/tenant | sim | Timeout de consulta (`AIOS-DB-0007`). |
| `db.query.idle_in_tx_timeout_ms` | int | 60000 | 1000–600000 | global | sim | Corta transações ociosas que seguram locks. |
| `db.query.slow_threshold_ms` | int | 500 | 10–60000 | global | sim | Limiar de captura de plano pelo `QueryGovernor`. |

### 1.2 Migração

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `db.migration.require_dry_run` | bool | `true` | true\|false | global | sim | Exige `DryRun` antes de `Applying` (invariante I6). |
| `db.migration.max_block_ms` | int | 200 | 0–5000 | global | sim | Bloqueio máximo tolerado (guarda T-03; NFR-010). |
| `db.migration.timeout_ms` | int | 900000 | 1000–3600000 | global | sim | Timeout total de aplicação (T-08). |
| `db.migration.lock_timeout_ms` | int | 5000 | 100–60000 | global | sim | Espera pelo advisory lock antes de `AIOS-MIGRATION-0001`. |

### 1.3 Isolamento

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `db.rls.enforce` | bool | `true` | true\|false | global | **não** | Desligar exige migração + aprovação explícita; auditado como violação P1. |

> **Aviso normativo.** `db.rls.enforce = false` **NÃO DEVE** ser usado em ambiente
> produtivo sob nenhuma circunstância. O parâmetro existe apenas para bootstrap de
> ambiente de desenvolvimento local, e sua alteração emite
> `aios._platform.database.compliance.violated`.

### 1.4 Partição e retenção

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `db.partition.precreate_ahead` | int | 7 | 1–90 | global/table | sim | Partições futuras mantidas criadas. |
| `db.partition.drop_grace_h` | int | 24 | 0–720 | global/table | sim | Janela entre `DETACH` e `DROP`. |
| `db.retention.default_ttl` | duration | `90d` | 1d–3650d | table | sim | TTL default quando a tabela não declara política. |
| `db.retention.batch_size` | int | 10000 | 100–1000000 | table | sim | Lote de `delete_batch`. |
| `db.retention.run_interval_min` | int | 60 | 5–1440 | global | sim | Frequência do `RetentionEnforcer`. |

### 1.5 Durabilidade e replicação

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `db.backup.full_interval_h` | int | 24 | 1–168 | global | sim | Periodicidade do backup full. |
| `db.backup.wal_archive_timeout_s` | int | 60 | 5–600 | global | sim | Força arquivamento de WAL — **limita o RPO** (NFR-009). |
| `db.backup.retention_days` | int | 35 | 7–3650 | global | sim | Retenção de artefatos em MinIO. |
| `db.backup.verify_interval_h` | int | 168 | 24–2160 | global | sim | Periodicidade do *restore drill* (FR-012). |
| `db.replication.max_lag_ms` | int | 1000 | 50–60000 | global | sim | Lag acima do qual a leitura em réplica é recusada (`AIOS-DB-0013`). |
| `db.replication.read_from_replica` | bool | `true` | true\|false | global/tenant | sim | Habilita roteamento de leitura elegível. |

### 1.6 Extensões

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `db.vector.default_index` | enum | `hnsw` | hnsw\|ivfflat\|none | global/table | sim | Tipo default de índice vetorial. |
| `db.vector.hnsw_m` | int | 16 | 4–96 | table | **não** | Parâmetro `m` do HNSW (alterar exige reindexação). |
| `db.vector.hnsw_ef_construction` | int | 64 | 16–512 | table | **não** | Qualidade de construção do HNSW. |
| `db.vector.ef_search` | int | 64 | 16–1024 | global/tenant | sim | Esforço de busca (troca recall × latência). |
| `db.graph.max_traversal_depth` | int | 6 | 1–20 | global/tenant | sim | Profundidade máxima de travessia AGE (contenção de DoS). |

### 1.7 Armazenamento e manutenção

| Chave | Tipo | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|------|---------|-------|--------|--------------|-----------|
| `db.storage.tenant_quota_gb` | int | 100 | 1–100000 | tenant | sim | Cota de armazenamento (limite efetivo vem de `../026-Cost-Optimizer/`). |
| `db.autovacuum.scale_factor_hot` | float | 0.02 | 0.001–0.5 | table | sim | Autovacuum agressivo em tabelas de alta rotatividade (NFR-016). |

---

## 2. Segredos (nunca em arquivo de configuração)

| Segredo | Origem | Rotação |
|---------|--------|---------|
| Credenciais das *roles* de banco (`<schema>_rw`, `platform_ddl`, `platform_admin`) | Cofre do `../021-Security/` | Automática, sem downtime (dupla credencial durante a janela). |
| Chave de acesso ao MinIO (WAL/backup) | Cofre do `../021-Security/` | Trimestral. |
| Certificados mTLS do serviço | PKI interna (`../021-Security/`) | Conforme política do `021`. |

Segredos **NÃO DEVEM** aparecer em variáveis de ambiente em texto claro nem em logs
(`./Logging.md` §5).

---

## 3. Precedência de Configuração

```
   valor efetivo = tabela (mais específico)
                 ▲ tenant
                 ▲ global
                 ▲ default do código (menos específico)
```

Uma chave de escopo `global` **NÃO PODE** ser sobrescrita por tenant; a tentativa é
rejeitada na validação de configuração. Chaves não recarregáveis alteradas em runtime
ficam pendentes até o próximo reinício e são reportadas na métrica
`aios_db_config_pending_reload`.

---

## 4. Exemplo — `appsettings.Production.json`

```json
{
  "Aios": {
    "Db": {
      "Pool":       { "MaxConnectionsPerService": 50, "Mode": "transaction" },
      "Query":      { "StatementTimeoutMs": 30000, "IdleInTxTimeoutMs": 60000, "SlowThresholdMs": 500 },
      "Migration":  { "RequireDryRun": true, "MaxBlockMs": 200, "TimeoutMs": 900000, "LockTimeoutMs": 5000 },
      "Rls":        { "Enforce": true },
      "Partition":  { "PrecreateAhead": 7, "DropGraceH": 24 },
      "Retention":  { "DefaultTtl": "90d", "BatchSize": 10000, "RunIntervalMin": 60 },
      "Backup":     { "FullIntervalH": 24, "WalArchiveTimeoutS": 60, "RetentionDays": 35, "VerifyIntervalH": 168 },
      "Replication":{ "MaxLagMs": 1000, "ReadFromReplica": true },
      "Vector":     { "DefaultIndex": "hnsw", "HnswM": 16, "HnswEfConstruction": 64, "EfSearch": 64 },
      "Graph":      { "MaxTraversalDepth": 6 },
      "Storage":    { "TenantQuotaGb": 100 },
      "Autovacuum": { "ScaleFactorHot": 0.02 }
    }
  }
}
```

Equivalente por ambiente:

```bash
export AIOS_DB_MIGRATION_REQUIRE_DRY_RUN=true
export AIOS_DB_MIGRATION_MAX_BLOCK_MS=200
export AIOS_DB_REPLICATION_MAX_LAG_MS=1000
export AIOS_DB_BACKUP_WAL_ARCHIVE_TIMEOUT_S=60
```

---

## 5. Relação Configuração → Requisito

| Chave | Requisito que sustenta |
|-------|------------------------|
| `db.migration.max_block_ms` | NFR-010 (migração sem downtime) |
| `db.migration.require_dry_run` | FR-004, invariante I6 |
| `db.rls.enforce` | FR-006, NFR-011 |
| `db.partition.precreate_ahead` | FR-007 |
| `db.retention.*` | FR-008, NFR-016 |
| `db.backup.wal_archive_timeout_s` | FR-011, NFR-009 (RPO) |
| `db.backup.verify_interval_h` | FR-012 |
| `db.replication.max_lag_ms` | FR-016, NFR-013 |
| `db.vector.*` | FR-013, NFR-003, NFR-006 |
| `db.graph.max_traversal_depth` | FR-014, mitigação de DoS |
| `db.pool.max_connections_per_service`, `db.query.statement_timeout_ms` | FR-015, NFR-004 |

---

## 6. Referências

- Brief: `./_DESIGN_BRIEF.md` §8
- Parâmetros do PostgreSQL: `./Database.md` §11
- Implantação: `./Deployment.md` · Segurança: `./Security.md`
