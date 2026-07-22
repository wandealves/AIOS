---
Documento: Logging
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0010, ADR-0052, ADR-0056
RFCs relacionados: RFC-0001
Depende de: 024-Observability, 025-Audit, Security.md, Metrics.md
---

# 005-Database — Log Estruturado

Stack: **Serilog → Seq**, com exportação OTel (`../024-Observability/`). Todo log é
**estruturado** (JSON); mensagens livres sem campos são defeito. Os campos de
correlação são os da RFC-0001 §5.6 e **não são redefinidos aqui**.

---

## 1. Campos Obrigatórios

| Campo | Origem | Obrigatório | Observação |
|-------|--------|-------------|------------|
| `timestamp` | serviço | sim | RFC 3339, UTC. |
| `level` | serviço | sim | `Verbose`\|`Debug`\|`Information`\|`Warning`\|`Error`\|`Fatal`. |
| `message_template` | Serilog | sim | Template, não a string interpolada (permite agregação). |
| `trace_id` / `span_id` | `traceparent` (W3C) | sim | Correlação com traces OTel. |
| `tenant_id` | `X-AIOS-Tenant` / sessão | sim em operação com tenant | `_platform` em operação de plataforma. |
| `service` | config | sim | `aios-database-svc`. |
| `component` | código | sim | `MigrationEngine`, `RetentionEnforcer`, … (PascalCase, igual a `./ClassDiagrams.md`). |
| `operation` | código | sim | `ApplyMigration`, `RunRetentionCycle`, … |
| `resource_urn` | requisição | quando aplicável | URN do alvo (RFC-0001 §5.1). |
| `error_code` | catálogo | em falha | `AIOS-DB-*` / `AIOS-MIGRATION-*`. |
| `duration_ms` | código | em operação concluída | Base para correlacionar com métricas. |
| `outcome` | código | sim | `ok`\|`denied`\|`failed`. |

---

## 2. Níveis — critério de uso

| Nível | Quando usar | Exemplos neste módulo |
|-------|-------------|-----------------------|
| `Verbose` | Detalhe de diagnóstico; **desligado** em produção. | Passo a passo de regra de lint. |
| `Debug` | Fluxo interno útil em investigação. | Escolha de réplica para leitura; plano capturado. |
| `Information` | Fato de negócio relevante e esperado. | Migração aplicada; partição criada; ciclo de retenção concluído. |
| `Warning` | Situação anômala que ainda não é falha. | Lag acima do limiar; fila de conexões; backup atrasado. |
| `Error` | Operação falhou e requer atenção. | Migração `Failed`; falha de arquivamento de WAL; expurgo interrompido. |
| `Fatal` | Serviço não pode continuar. | Impossível abrir conexão ao primário na inicialização. |

Um `Warning` que se repete sem ação é ruído: ou vira alerta com dono (`./Monitoring.md`)
ou é rebaixado a `Debug`.

---

## 3. Eventos de Log Canônicos

| `operation` | Nível | Campos específicos | Correspondência |
|-------------|-------|--------------------|-----------------|
| `SubmitMigration` | Information | `migration_version`, `owner_module`, `phase` | T-01 |
| `ValidateMigration` | Information / Warning | `violations[]`, `blocking_estimate_ms` | T-02 / T-03 |
| `DryRunMigration` | Information | `duration_ms`, `measured_block_ms` | T-04 |
| `ApplyMigration` | Information / Error | `lock_wait_ms`, `duration_ms`, `objects_affected[]` | T-06→T-07/T-08 |
| `RollbackMigration` | Warning | `reason`, `migration_version` | T-09 |
| `EnsurePartitions` | Information | `table`, `created[]`, `ahead_count` | UC-007 |
| `DropPartition` | Information | `partition`, `rows_removed`, `bytes_reclaimed` | UC-008 |
| `RunRetentionCycle` | Information | `tables_processed`, `rows_removed`, `skipped_legal_hold[]` | UC-008 |
| `ExecuteErasure` | Information | `request_urn`, `tables_affected[]`, `rows_erased`, `receipt_hash` | UC-009 |
| `AuditSchemas` | Information / Error | `violations[]`, `rls_coverage_ratio` | UC-006 |
| `RunBackup` / `VerifyBackup` | Information / Error | `kind`, `size_bytes`, `checksum`, `verified` | UC-010 |
| `RestorePointInTime` | Warning | `target_lsn`, `approvers[]`, `rto_ms` | UC-011 |
| `PromoteReplica` | Warning | `replica`, `lag_ms_at_promotion` | UC-012 |
| `TerminateQuery` | Warning | `pid`, `queryid`, `elapsed_ms` | UC-015 |
| `PolicyDecision` | Information / Warning | `capability`, `decision`, `deny_reason` | FR-017 |

---

## 4. Exemplos

### 4.1 Migração aplicada

```json
{
  "timestamp": "2026-07-22T14:03:47.912Z",
  "level": "Information",
  "message_template": "Migração {MigrationVersion} aplicada em {DurationMs}ms",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "tenant_id": "_platform",
  "service": "aios-database-svc",
  "component": "MigrationEngine",
  "operation": "ApplyMigration",
  "resource_urn": "urn:aios:_platform:migration:01J9ZB0K3N5Q7R9T1V3X5Z7A9C",
  "migration_version": "20260722T140000_add_memory_tag",
  "owner_module": "010-Memory",
  "phase": "expand",
  "lock_wait_ms": 12,
  "duration_ms": 1812,
  "objects_affected": ["memory.tag"],
  "outcome": "ok"
}
```

### 4.2 Migração reprovada no lint

```json
{
  "timestamp": "2026-07-22T13:40:02.114Z",
  "level": "Warning",
  "message_template": "Migração {MigrationVersion} reprovada: {ViolationCount} violações",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "tenant_id": "_platform",
  "component": "DdlConventionValidator",
  "operation": "ValidateMigration",
  "migration_version": "20260722T140000_add_memory_tag",
  "violations": ["RlsPolicyRequiredRule:memory.tag", "RetentionDeclaredRule:memory.tag"],
  "error_code": "AIOS-MIGRATION-0002",
  "outcome": "failed"
}
```

### 4.3 Expurgo concluído (sem PII no log)

```json
{
  "timestamp": "2026-07-22T14:22:09.441Z",
  "level": "Information",
  "message_template": "Expurgo {RequestUrn} concluído: {RowsErased} linhas",
  "trace_id": "9c1a2b3c4d5e6f708192a3b4c5d6e7f8",
  "tenant_id": "acme",
  "component": "ErasureCoordinator",
  "operation": "ExecuteErasure",
  "resource_urn": "urn:aios:acme:erasurerequest:01J9ZB3P6R8T0V2X4Z6B8D0F2H",
  "tables_affected": ["memory.item", "memory.embedding", "context.window", "graph.vertex"],
  "rows_erased": 18402,
  "receipt_hash": "sha256:1b8c…44df",
  "duration_ms": 41207,
  "outcome": "ok"
}
```

---

## 5. Regras de Privacidade no Log

| Regra | Detalhe |
|-------|---------|
| **Sem PII** | Logs **NÃO DEVEM** conter dados pessoais. Identificação é sempre por URN opaco (RFC-0001 §7). |
| **Sem segredos** | Credenciais, tokens, chaves de MinIO e strings de conexão **NÃO DEVEM** ser logados, nem mascarados parcialmente. |
| **Consulta normalizada** | O `QueryGovernor` registra o **texto normalizado** da consulta (`$1`, `$2`), nunca os valores dos parâmetros, e **nunca** consulta sobre tabela `pii`. |
| **Sem SQL de migração** | O log registra `up_sql_digest` e os objetos afetados, não o script completo (que pode conter dados em `UPDATE` de backfill). |
| **Sem conteúdo expurgado** | O log de expurgo registra contagens e hash do comprovante. |

Violações dessas regras são tratadas como incidente de segurança, não como defeito de
formatação.

---

## 6. Retenção e Amostragem

| Categoria | Nível | Retenção em Seq | Amostragem |
|-----------|-------|-----------------|------------|
| Operações administrativas | Information+ | 1 ano | 100% (nunca amostrado) |
| Conformidade e segurança | Warning+ | 1 ano | 100% |
| Fluxo interno | Debug | 7 dias | 100% em incidente, 1% em regime |
| Diagnóstico fino | Verbose | desligado em produção | — |

Log **não substitui auditoria**: a trilha imutável de ações privilegiadas vive em
`../025-Audit/` (`./Security.md` §9). O log é para diagnóstico; a auditoria é para
prova.

---

## 7. Correlação

```
   requisição HTTP ──traceparent──▶ span "ApplyMigration"
        │                              │
        │                              ├─ span "PolicyClient.Decide"     (022)
        │                              ├─ span "Postgres.ApplyDdl"
        │                              └─ span "Outbox.Enqueue"
        ▼
   log (trace_id, span_id) ──────▶ Seq          métrica (exemplar com trace_id) ──▶ Prometheus
                                     └────────── auditoria (trace_id) ──▶ 025-Audit
```

Um incidente é investigável partindo de qualquer um dos três sinais: o `trace_id` é o
elo comum.

---

## 8. Referências

- Métricas: `./Metrics.md` · Alertas: `./Monitoring.md`
- Segurança e auditoria: `./Security.md` · `../025-Audit/`
- Correlação (contrato): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6
