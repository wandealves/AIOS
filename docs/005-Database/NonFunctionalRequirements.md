---
Documento: NonFunctionalRequirements
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0005, ADR-0051, ADR-0053, ADR-0054, ADR-0056, ADR-0057, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: _DESIGN_BRIEF.md §7.2, 024-Observability, 027-Cluster, 035-Benchmark
---

# 005-Database — Requisitos Não-Funcionais

Metas derivadas de `./_DESIGN_BRIEF.md` §7.2. **Os valores numéricos abaixo são
normativos** e aparecem idênticos em `./Monitoring.md`, `./Benchmark.md`,
`./FailureRecovery.md`, `./Scalability.md` e `./Configuration.md`.

---

## 1. Tabela de Requisitos Não-Funcionais

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Desempenho (leitura por chave) | p99 ≤ **5 ms** para busca por PK/URN em tabela quente. | `aios_db_query_latency_ms{kind="pk"}` (histograma); carga sintética em `./Benchmark.md` §4. |
| NFR-002 | Desempenho (escrita transacional) | p99 ≤ **15 ms** para `INSERT`/`UPDATE` de linha única com commit durável. | `aios_db_query_latency_ms{kind="write"}`. |
| NFR-003 | Desempenho (busca vetorial) | p99 ≤ **50 ms** para top-K (K=10) em índice HNSW com ≥ **10⁷** vetores. | `aios_db_vector_search_latency_ms`; cenário V-1 em `./Benchmark.md`. |
| NFR-004 | Throughput | ≥ **20.000 transações/s** no primário (mix 80/20 leitura/escrita) com pooling. | `aios_db_transactions_total` (taxa); pgbench + carga sintética AIOS. |
| NFR-005 | Disponibilidade | ≥ **99,95%** do serviço de dados (primário + failover automático). | Uptime probe mensal; *error budget* de 21,9 min/mês. |
| NFR-006 | Qualidade de busca vetorial | Recall@10 ≥ **0,95** contra busca exata, por índice registrado. | Job de amostragem comparando HNSW vs. *exact scan*; `aios_db_vector_recall_ratio`. |
| NFR-007 | Escalabilidade | Suportar ≥ **10⁶** agentes e ≥ **10⁹** linhas em tabelas particionadas com crescimento sub-linear de latência. | Teste de escala em `./Scalability.md` §7. |
| NFR-008 | Durabilidade | Perda de transação commitada = **0** (`synchronous_commit=on` + réplica síncrona no quórum). | Chaos test: kill do primário sob carga; contagem de transações confirmadas × persistidas. |
| NFR-009 | Recuperação | **RTO ≤ 15 min**, **RPO ≤ 5 min** (alvo operacional de RPO ≤ **1 min** via WAL streaming). | DR drill trimestral; `aios_db_rpo_seconds` derivado do último WAL arquivado. |
| NFR-010 | Migração sem downtime | Migração `expand` NÃO DEVE bloquear escrita por mais de **200 ms**; `contract` executa apenas fora da janela de rollout. | `aios_db_migration_block_ms` (histograma); guarda T-03 da FSM. |
| NFR-011 | Isolamento multi-tenant | **0** vazamento entre tenants; **100%** das tabelas multi-tenant com RLS ativa. | Auditoria contínua (`POST /v1/database/schemas:audit`) + teste de penetração de RLS. |
| NFR-012 | Consistência | Isolamento `READ COMMITTED` como default, `REPEATABLE READ` disponível; OCC por `version` sem locks pessimistas no caminho quente. | Teste de concorrência; `aios_db_deadlocks_total` ≈ 0. |
| NFR-013 | Lag de replicação | p95 ≤ **500 ms**; leitura recusada acima de `db.replication.max_lag_ms` (default 1000 ms). | `aios_db_replication_lag_ms`. |
| NFR-014 | Observabilidade | **100%** das operações administrativas com trace OTel e correlação (`trace_id`, `tenant_id`). | Inspeção de spans; cobertura de campos obrigatórios em `./Logging.md`. |
| NFR-015 | Idempotência | Repetições de mutação administrativa produzem efeito único em **100%** dos casos. | Teste de replay com `Idempotency-Key` (RFC-0001 §5.5). |
| NFR-016 | Eficiência de armazenamento | *Bloat* de tabela ≤ **20%**; autovacuum ajustado por tabela de alta rotatividade. | `aios_db_table_bloat_ratio`. |

---

## 2. Orçamento de Erro e Priorização

| SLO | Orçamento mensal | Política ao esgotar |
|-----|------------------|---------------------|
| Disponibilidade 99,95% (NFR-005) | 21,9 min de indisponibilidade | Congelamento de migrações `contract` e de mudanças de configuração não-críticas até recomposição. |
| RPO ≤ 5 min (NFR-009) | 0 violações toleradas | Incidente P1 imediato; escrita entra em modo conservador se o arquivamento de WAL não progride. |
| Isolamento (NFR-011) | 0 violações toleradas | Incidente P1; migração emergencial reabilitando RLS; auditoria retroativa de acesso. |

Regra de priorização sob conflito: **integridade > isolamento > durabilidade >
disponibilidade > latência**. Quando duas metas colidem, a de maior precedência vence
e a decisão é registrada em `../025-Audit/`.

---

## 3. Atributos por Categoria

### 3.1 Desempenho
NFR-001 a NFR-004 definem o envelope operacional. O caminho quente de dados **NÃO
DEVE** atravessar o serviço `aios-database-svc`: consultas vão direto ao PostgreSQL
pelo `ConnectionPoolGateway` (ver `./Architecture.md` §1), evitando um salto de rede
por consulta.

### 3.2 Escalabilidade
NFR-007 é sustentado por particionamento (RANGE temporal + HASH por `tenant_id`),
retenção por *drop de partição* e réplicas de leitura. A degradação esperada com
crescimento de histórico é **sub-linear**, pois o *working set* permanece
proporcional ao presente (`./Scalability.md` §3).

### 3.3 Confiabilidade e Recuperação
NFR-008 e NFR-009 são verificados por *chaos test* e DR drill. Um backup não
verificado por *restore drill* (FR-012) **NÃO DEVE** ser considerado válido para fins
de RTO/RPO.

### 3.4 Segurança
NFR-011 é o invariante mais forte do módulo. Toda tabela com `multi_tenant = true`
**DEVE** ter RLS ativa; a divergência entre `schema_registry.rls_enabled` e o catálogo
real do PostgreSQL é tratada como violação P1 (`./Security.md` §6).

### 3.5 Manutenibilidade
NFR-010 e NFR-015 tornam a evolução previsível: toda migração passa por lint e
*dry-run*, e toda mutação administrativa é reexecutável sem efeito colateral.

### 3.6 Observabilidade
NFR-014 exige que 100% das operações administrativas emitam trace, métrica e log
correlacionados, conforme os campos obrigatórios da RFC-0001 §5.6.

---

## 4. Verificação — exemplo de consulta de SLI

```promql
# NFR-001 — p99 de leitura por chave nas últimas 24h
histogram_quantile(0.99,
  sum by (le) (rate(aios_db_query_latency_ms_bucket{kind="pk"}[24h]))
) < 5

# NFR-013 — p95 do lag de replicação
histogram_quantile(0.95,
  sum by (le) (rate(aios_db_replication_lag_ms_bucket[1h]))
) < 500

# NFR-009 — RPO observado (segundos desde o último WAL arquivado)
max(aios_db_rpo_seconds) < 300
```

---

## 5. Rastreabilidade

| NFR | Requisitos funcionais relacionados | Documento de verificação |
|-----|-----------------------------------|--------------------------|
| NFR-001, NFR-002, NFR-004 | FR-015 | `./Benchmark.md` |
| NFR-003, NFR-006 | FR-013 | `./Benchmark.md` §5 |
| NFR-005, NFR-008, NFR-009 | FR-011, FR-012 | `./FailureRecovery.md` |
| NFR-007, NFR-016 | FR-007, FR-008 | `./Scalability.md` |
| NFR-010, NFR-015 | FR-003, FR-004, FR-005 | `./Testing.md` |
| NFR-011 | FR-002, FR-006, FR-017 | `./Security.md` |
| NFR-012, NFR-013 | FR-016 | `./Testing.md` §7 |
| NFR-014 | FR-018 | `./Monitoring.md`, `./Logging.md`, `./Metrics.md` |

---

## 6. Referências

- Brief: `./_DESIGN_BRIEF.md` §7.2
- Requisitos funcionais: `./FunctionalRequirements.md`
- Monitoramento: `./Monitoring.md` · Métricas: `./Metrics.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
