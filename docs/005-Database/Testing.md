---
Documento: Testing
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0051, ADR-0052, ADR-0053, ADR-0055, ADR-0056, ADR-0057
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: Requirements.md, FunctionalRequirements.md, NonFunctionalRequirements.md
---

# 005-Database — Estratégia de Testes

Os identificadores de teste (`T-XXX-NN`) são os mesmos da matriz de rastreabilidade em
`./Requirements.md` §3.

---

## 1. Pirâmide de Testes

```
                    ┌────────────────────────┐
                    │  Chaos / DR (mensal)   │  kill do primário, PITR, drill
                    ├────────────────────────┤
                    │  Carga / Escala        │  20k tx/s, 10⁹ linhas, 10⁷ vetores
                    ├────────────────────────┤
                    │  E2E (CI noturno)      │  migração completa expand→contract
                    ├────────────────────────┤
                    │  Integração (por PR)   │  PG real em contêiner, RLS, partição
                    ├────────────────────────┤
                    │  Unidade (por commit)  │  regras de lint, FSM, políticas
                    └────────────────────────┘
```

Princípio: **testes de banco usam banco real**. Nenhuma suíte deste módulo usa SQLite,
banco em memória ou mock de driver — a semântica testada (RLS, DDL transacional,
advisory lock, partição) só existe no PostgreSQL.

---

## 2. Testes de Unidade

| ID | Alvo | Verifica |
|----|------|----------|
| T-UNIT-01 | `DdlConventionValidator` | Cada regra (CV-01..CV-12) isolada, com SQL positivo e negativo. |
| T-UNIT-02 | FSM de migração | Toda transição válida (T-01..T-10) e rejeição das inválidas (`AIOS-MIGRATION-0003`). |
| T-UNIT-03 | Cálculo de janela de retenção | Corte por `basis_column` + `ttl` com fusos e bordas. |
| T-UNIT-04 | Resolução de dimensão vetorial | Alteração de `dimensions` rejeitada (`AIOS-DB-0011`). |
| T-UNIT-05 | Precedência de configuração | table > tenant > global > default; escopo `global` não sobrescrito por tenant. |

Cobertura-alvo: **≥ 85%** de linhas nos pacotes `Schema`, `Migration` e `Lifecycle`.

---

## 3. Testes de Integração (PostgreSQL real)

| ID | Requisito | Cenário | Critério |
|----|-----------|---------|----------|
| T-CAT-01 | FR-001 | Aplicar migração que cria tabela → catálogo atualizado. | Linha presente com `owner_module` e `data_class`. |
| T-CAT-02 | FR-001 | Divergência forçada entre catálogo real e registry. | Auditoria reporta o desvio. |
| T-CAT-03 | FR-019 | `outbox` sem índice parcial. | Auditoria acusa violação. |
| T-CAT-04 | FR-020 | Popular tabela e rodar coleta. | `bytes_estimate` reflete o tamanho real (±10%). |
| T-MIG-01 | FR-002 | Tabela multi-tenant sem `tenant_id`. | `AIOS-MIGRATION-0002`. |
| T-MIG-02 | FR-002 | Tabela multi-tenant sem política RLS. | `AIOS-MIGRATION-0002`. |
| T-MIG-03 | FR-002 | DDL com `SECURITY DEFINER` / `DISABLE ROW LEVEL SECURITY`. | Rejeitado (CV-10). |
| T-MIG-04 | FR-003 | Aplicação normal. | Estado `Applied`; catálogo e outbox atualizados na mesma transação. |
| T-MIG-06 | FR-004 | `apply` sem `dry_run_at`. | `AIOS-MIGRATION-0007`. |
| T-MIG-07 | FR-005 | Rollback de migração reversível. | `RolledBack`; schema no estado anterior. |
| T-MIG-08 | FR-005 | Rollback de `contract` destrutiva. | `AIOS-MIGRATION-0006`. |
| T-PART-01 | FR-007 | Job de pré-criação. | ≥ `precreate_ahead` partições futuras. |
| T-PART-02 | FR-007 | Partição além do TTL. | `DETACH` + `DROP` após `drop_grace_h`. |
| T-RET-01 | FR-008 | Retenção em tabela `range_time`. | Remoção por `DROP`; `n_dead_tup` estável. |
| T-RET-02 | FR-008 | Retenção em tabela não particionada. | `delete_batch` respeita `batch_size` e `statement_timeout`. |
| T-ERA-01 | FR-009 | Expurgo de titular com dados em 4 tabelas + vetores. | `rows_erased` correto; `receipt_hash` presente. |
| T-ERA-02 | FR-009 | Auditoria do expurgo. | Registro em `025` com `trace_id` e comprovante. |
| T-ERA-03 | FR-010 | Expurgo sob `legal_hold`. | `AIOS-DB-0010`; **zero** linhas removidas. |
| T-VEC-01 | FR-013 | Criar índice HNSW e buscar top-10. | Índice usado no plano; resultado correto. |
| T-VEC-02 | FR-013 | Reindexação online sob escrita concorrente. | Sem bloqueio de escrita; recall mantido. |
| T-GRA-01 | FR-014 | Travessia no grafo do tenant A. | Nenhum vértice de B; profundidade > 6 rejeitada. |
| T-BKP-01 | FR-011 | Backup full + WAL. | Janela de PITR sem lacuna de LSN. |
| T-BKP-02 | FR-011 | Catálogo de backups. | `checksum` confere com o artefato em MinIO. |
| T-BKP-03 | FR-012 | *Restore drill*: restaurar backup em ambiente isolado e validar integridade. | `backup.verified` emitido; falha marca o backup como não verificado. |
| T-EVT-01 | FR-018 | Crash entre `COMMIT` e publicação. | Relay reenvia; consumidor deduplica por `event.id`. |

---

## 4. Testes de Contrato

| ID | Contrato | Verifica |
|----|----------|----------|
| T-CTR-01 | OpenAPI `/v1/database` | Requisição/resposta conformes ao schema publicado; erros no envelope RFC 7807 (RFC-0001 §5.4). |
| T-CTR-02 | `aios.database.v1` (proto) | Compatibilidade retroativa: nenhum campo removido ou renumerado. |
| T-CTR-03 | Eventos | Cada evento valida contra seu `dataschema`; campos desconhecidos toleram evolução aditiva. |
| T-CTR-04 | Idempotência (NFR-015) | Repetição de `apply`/`erasure` com a mesma `Idempotency-Key` → efeito único e resposta idêntica. |
| T-CTR-05 | Contrato do `outbox` | Instância em schema de domínio conforme `./Database.md` §3.9. |

---

## 5. Testes de Migração (gate de CI)

Toda migração submetida por qualquer módulo passa obrigatoriamente por:

1. **Lint** (`DdlConventionValidator`) — falha bloqueia o PR.
2. **Dry-run** em réplica com o volume de *staging*, medindo `duration_ms` e bloqueio.
3. **Verificação de orçamento**: `measured_block_ms` ≤ `db.migration.max_block_ms`
   (200 ms, NFR-010) — acima disso, o PR é rejeitado com a orientação de reescrever
   como expand/contract.
4. **T-MIG-05** — teste de concorrência: dois *runners* aplicam simultaneamente; o
   segundo **DEVE** receber `AIOS-MIGRATION-0001` (invariante I1).

```bash
# gate local antes de abrir PR
aios db migration lint    ./migrations/20260722T140000_add_memory_tag.sql
aios db migration dry-run ./migrations/20260722T140000_add_memory_tag.sql --env staging
# → duration=1.8s block=40ms  ✔ dentro do orçamento (200ms)
```

---

## 6. Testes de Padrão de Acesso (modo `transaction` do PgBouncer)

| ID | Verifica |
|----|----------|
| T-CONN-01 | Cota de conexões por serviço: o serviço A saturando o pool **não** afeta o B; excesso → `AIOS-DB-0006`. |
| T-CONN-02 | Consulta acima de `statement_timeout` é abortada com `AIOS-DB-0007` e libera a conexão. |
| T-CONN-03 | Nenhum código do AIOS depende de estado de sessão (prepared statement nomeado, `SET` persistente, `LISTEN/NOTIFY`) — incompatível com `pool_mode=transaction`. |

T-CONN-03 é o teste que mais previne incidentes sutis: o padrão funciona em `dev`
(pool `session`) e falha em produção. Por isso `dev` também roda em modo `transaction`
no CI.

---

## 7. Testes de Concorrência e Replicação

| ID | Requisito | Cenário | Critério |
|----|-----------|---------|----------|
| T-CONC-01 | NFR-012 | 100 escritores concorrentes na mesma linha com OCC. | Conflitos retornam `AIOS-DB-0004`; nenhuma perda de atualização; `aios_db_deadlocks_total` = 0. |
| T-REPL-01 | FR-016, NFR-013 | Lag induzido acima do limiar. | Réplica sai do pool de leitura; `AIOS-DB-0013` ou redirecionamento ao primário. |
| T-REPL-02 | NFR-008 | Kill do primário com transações em voo. | Nenhuma transação **confirmada** ao cliente é perdida. |

---

## 8. Testes de Segurança

| ID | Requisito | Cenário | Critério |
|----|-----------|---------|----------|
| T-SEC-01 | FR-006, NFR-011 | Pentest de RLS: tentativa de leitura cruzada por SQL malicioso e por omissão de `app.tenant_id`. | Zero linhas de outro tenant; sessão sem tenant vê zero linhas. |
| T-SEC-02 | STRIDE-S | Serviço tenta conectar com *role* de outro schema. | Conexão recusada; tentativa auditada. |
| T-SEC-03 | FR-017 | Operação administrativa sem capability; PDP indisponível. | `AIOS-DB-0002` em ambos os casos (*fail-closed*). |
| T-SEC-04 | STRIDE-T | Tentativa de `UPDATE` em `platform.migration` em estado terminal. | Rejeitado; alerta de conformidade. |
| T-SEC-05 | LGPD | Log e evento de expurgo. | Nenhum campo com PII; apenas URN, contagem e hash. |

---

## 9. Testes de Carga e Escala

| ID | Requisito | Carga | Critério |
|----|-----------|-------|----------|
| T-LOAD-01 | NFR-004 | Mix 80/20 leitura/escrita a 20.000 tx/s por 30 min. | p99 dentro de NFR-001/NFR-002; sem erro sustentado. |
| T-LOAD-02 | NFR-003, NFR-006 | 10⁷ vetores, 500 buscas/s top-10. | p99 ≤ 50 ms; Recall@10 ≥ 0,95. |
| T-SCALE-01 | NFR-007 | 10⁹ linhas em tabela particionada, 90 dias de histórico. | Latência de leitura por chave estável (degradação sub-linear). |
| T-SCALE-02 | NFR-016 | Ciclo completo de retenção sob carga. | `bloat` ≤ 20%; sem impacto perceptível em p99. |

Metodologia detalhada em `./Benchmark.md`.

---

## 10. Testes de Caos e DR

| ID | Cenário | Critério |
|----|---------|----------|
| T-CHAOS-01 | Kill do primário sob carga. | Failover; **0** transação commitada perdida (NFR-008); escrita restabelecida dentro do RTO. |
| T-CHAOS-02 | MinIO indisponível por 30 min. | Plano de dados segue operando; alerta de RPO degradado; WAL drena ao restabelecer. |
| T-CHAOS-03 | Partição de rede com o NATS. | Eventos acumulam em `platform.outbox`; nenhum evento perdido; backlog drena. |
| T-CHAOS-04 | PDP (`022`) indisponível. | Operações administrativas negadas (*fail-closed*); plano de dados intacto. |
| T-DR-01 | DR drill trimestral: PITR completo a partir do backup. | **RTO ≤ 15 min**, **RPO ≤ 5 min** (NFR-009). |
| T-DR-02 | Restauração parcial de um único tenant. | Dados do tenant restaurados sem afetar os demais. |

---

## 11. Ambientes, Fixtures e Gates

| Ambiente | Uso | Dados |
|----------|-----|-------|
| CI efêmero | unidade, integração, contrato | PostgreSQL 16 em contêiner com `pgvector` e `age`; *fixtures* sintéticas por tenant (`acme`, `globex`). |
| `staging` | dry-run de migração, carga reduzida | Volume ~10% da produção, dados anonimizados. |
| `prod` | drills de DR e verificação de backup | Dados reais; operações somente de leitura/verificação. |

**Gates de merge:**
1. Unidade + integração verdes.
2. Lint de migração sem violação bloqueante.
3. Dry-run dentro do orçamento de bloqueio (200 ms).
4. Testes de contrato sem quebra retroativa.
5. Cobertura ≥ 85% nos pacotes críticos.

Sem estes cinco, o PR **NÃO DEVE** ser mesclado.

---

## 12. Referências

- Rastreabilidade: `./Requirements.md` §3
- Requisitos: `./FunctionalRequirements.md` · `./NonFunctionalRequirements.md`
- Carga e KPIs: `./Benchmark.md` · Falhas: `./FailureRecovery.md`
