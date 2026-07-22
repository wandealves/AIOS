---
Documento: FunctionalRequirements
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0050, ADR-0051, ADR-0052, ADR-0053, ADR-0054, ADR-0055, ADR-0056, ADR-0057, ADR-0058, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: _DESIGN_BRIEF.md §7.1, 022-Policy, 025-Audit, 026-Cost-Optimizer, 027-Cluster
---

# 005-Database — Requisitos Funcionais

Requisitos derivados de `./_DESIGN_BRIEF.md` §7.1. Prioridade em **MoSCoW**
(Must / Should / Could). Palavras normativas conforme RFC 2119. Todo FR é
verificável por ao menos um caso de teste em `./Testing.md`.

---

## 1. Tabela de Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de aceite | Origem |
|----|-----------|-----------|--------------------|--------|
| FR-001 | O módulo DEVE manter o catálogo autoritativo de schemas, tabelas, donos e classificação de dados em `platform.schema_registry`. | Must | `GET /v1/database/schemas` lista 100% das tabelas de domínio com `owner_module` e `data_class` preenchidos; divergência com o catálogo real do PostgreSQL é reportada. | Brief §1.2 R-01, §3.1 |
| FR-002 | O módulo DEVE rejeitar a criação de tabela multi-tenant sem `tenant_id`, sem política RLS ou sem retenção declarada. | Must | Migração violadora → `Failed` com `AIOS-MIGRATION-0002`; teste de lint cobre os três casos isoladamente. | Brief §1.2 R-02/R-03 |
| FR-003 | O módulo DEVE aplicar migrações forward-only sob *advisory lock* global, uma por vez em todo o cluster. | Must | Duas aplicações concorrentes: a segunda recebe `AIOS-MIGRATION-0001`; `platform.migration` nunca tem duas linhas em `Applying`. | Brief §4 I1, ADR-0053 |
| FR-004 | O módulo DEVE exigir *dry-run* bem-sucedido em réplica antes da aplicação, quando `db.migration.require_dry_run = true`. | Must | `apply` sem `dry_run_at` → `AIOS-MIGRATION-0007`. | Brief §4 I6 |
| FR-005 | O módulo DEVERIA permitir reverter apenas migrações marcadas como reversíveis e sem migrações posteriores dependentes. | Should | Rollback de migração `contract` destrutiva → `AIOS-MIGRATION-0006`. | Brief §4 T-09, I3 |
| FR-006 | O módulo DEVE garantir isolamento multi-tenant por RLS em toda tabela multi-tenant. | Must | Sessão com `tenant_id` A não lê nenhuma linha de B, verificado em 100% das tabelas com `multi_tenant = true`. | Brief §1.2 R-03, §12.1 |
| FR-007 | O módulo DEVE pré-criar partições futuras e remover as expiradas conforme `platform.partition_policy`. | Must | Sempre existem ≥ `precreate_ahead` partições futuras; nenhuma partição além do TTL sobrevive a um ciclo do enforcer. | Brief §1.2 R-05 |
| FR-008 | O módulo DEVE executar retenção preferindo *drop de partição* a `DELETE` em tabelas particionadas por tempo. | Must | Tabelas `range_time` expurgam sem crescimento de tuplas mortas (`pg_stat_user_tables.n_dead_tup` estável). | Brief §1.2 R-06, ADR-0055 |
| FR-009 | O módulo DEVE executar o direito ao esquecimento com comprovante rastreável. | Must | `aios.<tenant>.database.erasure.completed` emitido com `rows_erased` e `receipt_hash` registrado em `../025-Audit/`. | Brief §12.3 |
| FR-010 | O módulo DEVE suspender qualquer expurgo — inclusive automático por TTL — sob `legal_hold` ativo. | Must | Pedido durante *hold* → `AIOS-DB-0010`; nenhuma linha removida; conflito registrado. | Brief §12.3 |
| FR-011 | O módulo DEVE manter WAL archiving contínuo e catálogo de backups com janela de PITR sem lacuna. | Must | `GET /v1/database/backups` informa janela contínua de LSN; ausência de lacuna verificada por job. | Brief §1.2 R-07 |
| FR-012 | O módulo DEVE verificar backups por *restore drill* automatizado periódico. | Must | `backup.verified` emitido a cada `db.backup.verify_interval_h`; falha gera alerta P1 e novo backup full. | Brief §1.2 R-07, P-05 |
| FR-013 | O módulo DEVE governar índices vetoriais (tipo, parâmetros, operador de distância) e permitir reindexação online. | Must | `PutVectorIndex` cria o índice; `:reindex` executa `REINDEX ... CONCURRENTLY` sem bloquear escrita. | Brief §1.2 R-08, §3.7 |
| FR-014 | O módulo DEVERIA governar grafos Apache AGE com isolamento por tenant equivalente ao da RLS. | Should | Travessia no grafo do tenant A não retorna vértices/arestas de B. | Brief §1.2 R-08 |
| FR-015 | O módulo DEVE aplicar cota de conexões e `statement_timeout` por serviço chamador. | Must | Excesso de conexões → `AIOS-DB-0006`; consulta acima do timeout → `AIOS-DB-0007`. | Brief §1.2 R-09 |
| FR-016 | O módulo DEVERIA recusar leitura em réplica cujo lag exceda `db.replication.max_lag_ms`. | Should | Lag acima do limiar → `AIOS-DB-0013` ou redirecionamento transparente ao primário. | Brief §1.2 R-09, §10 |
| FR-017 | O módulo DEVE consultar o PDP do `022-Policy` antes de toda operação administrativa, com *default deny*. | Must | Operação sem capability → `AIOS-DB-0002`; auditoria registra a decisão `deny`. | Brief §1.2 R-11, RFC-0001 §5.8 |
| FR-018 | O módulo DEVE emitir todos os eventos de `./Events.md` via Outbox transacional. | Must | Nenhum evento perdido em teste de crash entre `COMMIT` e publicação. | Brief §1.2 R-12 |
| FR-019 | O módulo DEVERIA publicar o contrato canônico da tabela `outbox` e validar suas instâncias nos schemas de domínio. | Should | Auditoria detecta `outbox` sem índice parcial `WHERE published = false` ou sem política de retenção. | Brief §3.8 |
| FR-020 | O módulo DEVERIA reportar consumo de armazenamento por tenant como insumo de custo. | Should | `bytes_estimate` atualizado em ≤ 24h; evento `storage.threshold` emitido ao cruzar o limiar. | Brief §1.2 R-01, `../026-Cost-Optimizer/` |

---

## 2. Regras de Negócio Transversais

| # | Regra | Aplicação |
|---|-------|-----------|
| RN-01 | Toda mutação administrativa **DEVE** aceitar `Idempotency-Key` e produzir efeito único (RFC-0001 §5.5). | FR-003, FR-005, FR-009, FR-017 |
| RN-02 | Toda operação administrativa **DEVE** ser auditada em `../025-Audit/` com `trace_id`, identidade e resultado. | FR-017, FR-009, FR-012 |
| RN-03 | Nenhum documento, script ou operação **PODE** redefinir contratos centrais (URN, envelope de evento/erro, correlação, subjects) — eles vêm da RFC-0001. | Todos |
| RN-04 | Estados terminais de `platform.migration` são imutáveis, exceto pelo preenchimento de `superseded_by`. | FR-003, FR-005 |
| RN-05 | A obrigação de preservar (`legal_hold`) **prevalece** sobre a de apagar (TTL/esquecimento); o conflito é registrado, nunca resolvido silenciosamente. | FR-010 |

---

## 3. Exemplo de Verificação — FR-002

```bash
# Submete migração que cria tabela multi-tenant SEM política de RLS
curl -sX POST https://api.aios.local/v1/database/migrations \
  -H "X-AIOS-Tenant: _platform" \
  -H "Idempotency-Key: 01J9ZB0K3N5Q7R9T1V3X5Z7A9C" \
  -H "Content-Type: application/json" \
  -d '{
        "version": "20260722T140000_add_memory_tag",
        "ownerModule": "010-Memory",
        "phase": "expand",
        "upSql": "CREATE TABLE memory.tag (urn text PRIMARY KEY, tenant_id text NOT NULL, name text NOT NULL);"
      }'

# → 201 Created, estado Draft
curl -sX POST https://api.aios.local/v1/database/migrations/{urn}:validate

# → 422 Unprocessable Entity
# {
#   "code": "AIOS-MIGRATION-0002",
#   "title": "Migration Rejected by DDL Convention Lint",
#   "detail": "memory.tag: tabela multi-tenant sem POLICY de RLS e sem retenção declarada.",
#   "retriable": false
# }
```

---

## 4. Rastreabilidade

A matriz FR → UC → Teste está consolidada em `./Requirements.md` §3. Cada FR aparece
literalmente, com o mesmo identificador, em `./UseCases.md`, `./Testing.md` e — quando
tem meta numérica associada — em `./NonFunctionalRequirements.md`.

---

## 5. Referências

- Brief (fonte única de verdade): `./_DESIGN_BRIEF.md` §7.1
- Requisitos não-funcionais: `./NonFunctionalRequirements.md`
- Casos de uso: `./UseCases.md` · Testes: `./Testing.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
