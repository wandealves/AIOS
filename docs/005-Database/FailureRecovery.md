---
Documento: FailureRecovery
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0053, ADR-0055, ADR-0056, ADR-0059
RFCs relacionados: RFC-0001
Depende de: _DESIGN_BRIEF.md §9, 027-Cluster, 029-Operations, Monitoring.md
---

# 005-Database — Falhas e Recuperação

Metas: **RTO ≤ 15 min**, **RPO ≤ 5 min** (alvo operacional ≤ 1 min via WAL streaming
contínuo), alinhadas a V-R10 da Visão Global e a NFR-009.

Princípio de degradação: sob falha parcial, o módulo prioriza **preservar a integridade
e negar com segurança** — rejeitar escrita, recusar leitura obsoleta, *fail-closed* no
PEP — em vez de servir dado incorreto ou sem governança.

---

## 1. FMEA — Modos de Falha

| # | Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry | Severidade |
|---|---------------|----------|-----------|-------------|--------------------|-----------|
| F-01 | Perda do primário PostgreSQL | Health check + ausência de heartbeat de replicação | Escritas rejeitadas (`AIOS-DB-0005`); leituras seguem em réplica | Promoção da réplica de menor lag, coordenada com `../027-Cluster/`; PgBouncer reconfigurado | Escritas não confirmadas são reenviadas pelo cliente com `Idempotency-Key` | **P1** |
| F-02 | Réplica com lag excessivo | `aios_db_replication_lag_ms` > `max_lag_ms` | Réplica removida do pool de leitura | Recuperação natural ou *rebuild*; evento `replication.lagging` | Leitura redirecionada ao primário (`AIOS-DB-0013`) | P2 |
| F-03 | Migração falha no meio | Erro/timeout em `Applying` | DDL transacional reverte tudo (invariante I2) | `Failed`; correção por **nova** migração (forward-fix, T-10) | Lock liberado; reaplicação idempotente por `version` | P2 |
| F-04 | Migração trava tabela além do orçamento | `aios_db_migration_block_ms` > 200 | Abortada por `lock_timeout` | Reescrita como expand/contract incremental | Nova `version`, sem efeito parcial | P2 |
| F-05 | Esgotamento do pool de conexões | Fila do PgBouncer > 0 sustentada | Bulkhead por serviço: um serviço não drena o pool dos outros | `AIOS-DB-0006` com `Retry-After`; escalar réplicas de leitura | Retry com backoff exponencial | P2 |
| F-06 | Consulta patológica (plano regredido) | `pg_stat_statements` + captura de plano | `statement_timeout` aborta | `QueryGovernor` sinaliza índice ausente; `ANALYZE`/reindex | Consulta de leitura: retry seguro | P3 |
| F-07 | Disco cheio no primário | Alerta de espaço + falha de WAL | Escritas bloqueadas **antes** da corrupção | Expurgo de partições expiradas + expansão de volume; `AIOS-DB-0012` | Transação falha inteira, sem efeito parcial | **P1** |
| F-08 | Corrupção de página / perda de dados | Checksums (`--data-checksums`) + falha de leitura | Tabela/partição isolada | **PITR** até o LSN saudável | Restauração determinística por LSN | **P1** |
| F-09 | Falha de arquivamento de WAL | `aios_db_rpo_seconds` crescendo | Alerta imediato — o RPO está degradado | Reenvio a MinIO; se persistir, escrita entra em modo conservador | Segmentos de WAL são idempotentes por nome | **P1** |
| F-10 | Backup irrecuperável | *Restore drill* falha (FR-012) | Backup marcado como não verificado | Novo backup full imediato + investigação | Verificação repetível | **P1** |
| F-11 | Crash entre commit e publish de evento | Relay detecta `published = false` | Estado no PostgreSQL intacto | Relay reenvia; consumidores deduplicam por `event.id` | Exactly-once efetivo | P3 |
| F-12 | RLS desabilitada indevidamente | Auditoria contínua (`schemas:audit`) | Evento `compliance.violated` | Reabilitação imediata por migração emergencial; auditoria retroativa de acessos | `ENABLE ROW LEVEL SECURITY` é idempotente | **P1** |
| F-13 | PDP (`022`) indisponível | Timeout/CB no `PolicyClient` | Bulkhead do PEP | *Fail-closed*: operação administrativa negada (`AIOS-DB-0002`) | Retry após meia-abertura do CB | P2 |
| F-14 | Expurgo interrompido no meio | `erasure_request` em `executing` além do prazo | Lote parcial já commitado, em ordem segura de FK | Retomada idempotente pelo restante do `subject_urn` | Remover o inexistente é operação nula | P2 |
| F-15 | Partição futura ausente ao virar o período | `aios_db_partitions_ahead` < mínimo | Escrita da tabela falharia | `POST /v1/database/partitions:ensure` imediato | Criação idempotente | **P1** |
| F-16 | NATS indisponível | Health do `EventEmitter` | Outbox retém eventos; plano de dados intacto | Backlog drena ao religar | Ordenado por stream; dedupe por `event.id` | P3 |

---

## 2. Detecção — Sinais Primários

| Sinal | Fonte | Falhas que revela |
|-------|-------|-------------------|
| `up{job="aios-postgres-primary"}` | Prometheus | F-01 |
| `aios_db_rpo_seconds` | serviço | F-09, F-10 |
| `aios_db_replication_lag_ms` | serviço | F-02, precursor de F-01 |
| `aios_db_pool_connections_waiting` | PgBouncer | F-05 |
| `aios_db_rls_coverage_ratio` | auditoria | F-12 |
| `aios_db_partitions_ahead` | job de partição | F-15 |
| `aios_db_outbox_pending` | relay | F-11, F-16 |
| Espaço livre no volume | nó | F-07 |

---

## 3. Isolamento e Bulkheads

```
   ┌── serviço 006 ──┐   ┌── serviço 010 ──┐   ┌── serviço 015 ──┐
   │ pool 50 conexões│   │ pool 50 conexões│   │ pool 50 conexões│  ← bulkhead
   └────────┬────────┘   └────────┬────────┘   └────────┬────────┘
            └─────────────────────┼─────────────────────┘
                          PgBouncer (transaction)
                                  │
                         PostgreSQL primário
                                  │
              ┌───────────────────┼────────────────────┐
        statement_timeout   max_traversal_depth   tenant_quota_gb
         (query longa)        (grafo explosivo)     (disco por tenant)
```

Cada linha de defesa contém uma classe distinta de falha: contenção de **conexão**,
de **tempo**, de **profundidade** e de **espaço**. Nenhuma delas depende do bom
comportamento do chamador.

---

## 4. Estratégia de Retry

| Erro | `retriable` | Política do cliente |
|------|-------------|---------------------|
| `AIOS-DB-0005` (primário indisponível) | sim | Backoff exponencial 100 ms → 5 s, jitter, máx. 6 tentativas. |
| `AIOS-DB-0006` (pool esgotado) | sim | Respeitar `Retry-After`; não aumentar concorrência. |
| `AIOS-DB-0007` (statement timeout) | sim | Repetir **uma** vez; se recorrer, é problema de plano/índice, não de tentativa. |
| `AIOS-DB-0013` (lag alto) | sim | Repetir apontando ao primário. |
| `AIOS-DB-0004` (conflito OCC) | sim | Reler, reaplicar, repetir (máx. 5). |
| `AIOS-MIGRATION-0001` (lock ocupado) | sim | Aguardar e repetir; nunca forçar. |
| `AIOS-DB-0002`, `-0003`, `-0010`, `-0011` | **não** | Corrigir a causa; repetir não resolve. |

Toda mutação repetida **DEVE** reusar a mesma `Idempotency-Key` (RFC-0001 §5.5).

---

## 5. Procedimentos de Recuperação

### 5.1 Failover do primário (F-01)

```
 1. Confirmar indisponibilidade real (não partição de rede do observador).
 2. 027-Cluster publica cluster.failover.requested.
 3. ReplicationSupervisor escolhe a réplica de menor lag e a promove.
 4. PgBouncer reaponta para o novo primário; escrita restabelecida.
 5. Réplica antiga é reconstruída a partir do novo primário.
 6. Emitir database.replication.promoted e registrar o RPO efetivo.
```
Alvo: escrita restabelecida em **≤ 5 min**; total dentro de RTO ≤ 15 min.

### 5.2 PITR após corrupção ou erro humano (F-08)

```
 1. Congelar escrita; declarar incidente.
 2. Escolher alvo (timestamp ou LSN) imediatamente anterior ao dano.
 3. POST /v1/database/restore (capability + 2ª aprovação).
 4. Restaurar último full anterior ao alvo; reaplicar WAL até o LSN.
 5. Validar integridade (checksums, contagens por tabela crítica).
 6. Reabrir escrita; registrar RTO medido em 025-Audit.
```

### 5.3 Reabilitação de RLS (F-12)

```
 1. Identificar a tabela pelo evento compliance.violated.
 2. Migração emergencial: ENABLE + FORCE ROW LEVEL SECURITY + POLICY.
 3. Auditar acessos ocorridos na janela sem proteção (25-Audit).
 4. Post-mortem obrigatório: como o DDL passou pelo lint?
```

---

## 6. Testes de Recuperação

| Exercício | Frequência | Critério |
|-----------|-----------|----------|
| *Restore drill* automatizado | Semanal (`db.backup.verify_interval_h` = 168) | Backup restaura e valida; `backup.verified` emitido. |
| Chaos: kill do primário sob carga | Mensal | 0 transação commitada perdida (NFR-008). |
| DR drill completo (PITR de 500 GiB) | Trimestral | RTO ≤ 15 min, RPO ≤ 5 min. |
| Restauração parcial de um tenant | Trimestral | Tenant restaurado sem afetar os demais. |
| Simulação de PDP indisponível | Trimestral | Operações administrativas negadas; plano de dados intacto. |

Um backup **nunca verificado** é tratado como inexistente (FR-012): a métrica que
importa não é "temos backup", é "restauramos com sucesso na última semana".

---

## 7. Degradação Graciosa — Ordem de Sacrifício

Quando não é possível manter tudo, o módulo sacrifica nesta ordem:

```
   1º  Consultas analíticas / relatórios          (menor impacto operacional)
   2º  Leituras em réplica com lag                (redireciona ao primário)
   3º  Operações administrativas não urgentes     (migração, reindexação)
   4º  Escritas não críticas (telemetria, outbox de baixa prioridade)
   ─────────────────── nunca sacrificado ───────────────────
   ·  Integridade transacional
   ·  Isolamento entre tenants (RLS)
   ·  Durabilidade de transação confirmada
```

---

## 8. Referências

- Brief: `./_DESIGN_BRIEF.md` §9
- Alertas e runbooks: `./Monitoring.md` · `../029-Operations/`
- HA/DR do cluster: `../027-Cluster/` · Testes: `./Testing.md` §10
- Erros: `./API.md` §5
