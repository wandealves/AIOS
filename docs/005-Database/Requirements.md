---
Documento: Requirements
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0050, ADR-0051, ADR-0052, ADR-0053, ADR-0054, ADR-0055, ADR-0056, ADR-0057, ADR-0058, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: FunctionalRequirements.md, NonFunctionalRequirements.md, UseCases.md, Testing.md
---

# 005-Database — Índice de Requisitos e Rastreabilidade

Este documento consolida os requisitos do módulo e estabelece a **matriz de
rastreabilidade** Requisito → Caso de Uso → Teste. Ele não introduz requisitos novos:
os FR estão em `./FunctionalRequirements.md` e os NFR em
`./NonFunctionalRequirements.md`, ambos derivados de `./_DESIGN_BRIEF.md` §7.

---

## 1. Stakeholders e Interesses

| Stakeholder | Interesse primário | Requisitos que o representam |
|-------------|--------------------|------------------------------|
| Engenheiro de módulo (`006`–`026`) | Evoluir seu schema sem quebrar produção. | FR-001..FR-005, NFR-010 |
| SRE / Operador de plataforma | Durabilidade, recuperação e latência previsíveis. | FR-011, FR-012, FR-015, FR-016, NFR-005, NFR-008, NFR-009, NFR-013 |
| Arquiteto de segurança | Isolamento entre tenants e privilégio mínimo. | FR-002, FR-006, FR-017, NFR-011 |
| Encarregado de dados (DPO) | Conformidade LGPD/GDPR verificável. | FR-009, FR-010, NFR-011 |
| Engenheiro de custo (`026`) | Atribuição de custo de armazenamento. | FR-020, NFR-016 |
| Time de conhecimento (`010`, `018`, `019`) | Busca vetorial e grafo com qualidade previsível. | FR-013, FR-014, NFR-003, NFR-006 |
| Auditoria (`025`) | Trilha completa de operações privilegiadas. | FR-009, FR-017, FR-018, NFR-014 |

---

## 2. Índice Consolidado

### 2.1 Requisitos Funcionais (20)

| Faixa | Tema | IDs |
|-------|------|-----|
| Catálogo e convenções | Registro de schemas, lint de DDL | FR-001, FR-002, FR-019 |
| Migração | Submissão, ensaio, aplicação, rollback | FR-003, FR-004, FR-005 |
| Isolamento | RLS multi-tenant | FR-006 |
| Ciclo de vida do dado | Partição, retenção, expurgo, legal hold | FR-007, FR-008, FR-009, FR-010 |
| Durabilidade | Backup, WAL, PITR, verificação | FR-011, FR-012 |
| Extensões | pgvector, Apache AGE | FR-013, FR-014 |
| Acesso e contenção | Pooling, timeout, réplica | FR-015, FR-016 |
| Governança | PEP→PDP, eventos, custo | FR-017, FR-018, FR-020 |

### 2.2 Requisitos Não-Funcionais (16)

| Categoria | IDs |
|-----------|-----|
| Desempenho | NFR-001, NFR-002, NFR-003, NFR-004 |
| Disponibilidade e recuperação | NFR-005, NFR-008, NFR-009 |
| Qualidade de busca | NFR-006 |
| Escalabilidade e eficiência | NFR-007, NFR-016 |
| Manutenibilidade | NFR-010, NFR-015 |
| Segurança | NFR-011 |
| Consistência e replicação | NFR-012, NFR-013 |
| Observabilidade | NFR-014 |

### 2.3 Casos de Uso (15)

`UC-001` a `UC-015`, especificados em `./UseCases.md`.

---

## 3. Matriz de Rastreabilidade — FR → UC → Teste

| FR | Caso(s) de uso | Caso(s) de teste (`./Testing.md`) | NFR associado |
|----|----------------|-----------------------------------|---------------|
| FR-001 | UC-001, UC-006 | T-CAT-01, T-CAT-02 | NFR-014 |
| FR-002 | UC-002 | T-MIG-01, T-MIG-02, T-MIG-03 | NFR-011 |
| FR-003 | UC-004 | T-MIG-04, T-MIG-05 (concorrência) | NFR-010 |
| FR-004 | UC-003, UC-004 | T-MIG-06 | NFR-010 |
| FR-005 | UC-005 | T-MIG-07, T-MIG-08 | NFR-015 |
| FR-006 | UC-006 | T-SEC-01 (penetração RLS), T-SEC-02 | NFR-011 |
| FR-007 | UC-007 | T-PART-01, T-PART-02 | NFR-007 |
| FR-008 | UC-008 | T-RET-01, T-RET-02 | NFR-016 |
| FR-009 | UC-009 | T-ERA-01, T-ERA-02 | NFR-014 |
| FR-010 | UC-009 | T-ERA-03 (legal hold) | — |
| FR-011 | UC-010, UC-011 | T-BKP-01, T-BKP-02 | NFR-009 |
| FR-012 | UC-010 | T-BKP-03 (restore drill) | NFR-009 |
| FR-013 | UC-013 | T-VEC-01, T-VEC-02 | NFR-003, NFR-006 |
| FR-014 | UC-013 | T-GRA-01 | NFR-011 |
| FR-015 | UC-014 | T-CONN-01, T-CONN-02 | NFR-004 |
| FR-016 | UC-012, UC-014 | T-REPL-01 | NFR-013 |
| FR-017 | UC-002, UC-004, UC-009, UC-011 | T-SEC-03 (default deny) | NFR-011 |
| FR-018 | UC-004, UC-008, UC-010 | T-EVT-01 (crash entre commit e publish) | NFR-014 |
| FR-019 | UC-006 | T-CAT-03 | NFR-016 |
| FR-020 | UC-006 | T-CAT-04 | NFR-016 |

---

## 4. Matriz NFR → Método de Verificação

| NFR | Método | Documento |
|-----|--------|-----------|
| NFR-001, NFR-002, NFR-004 | Teste de carga (pgbench + carga sintética AIOS) | `./Benchmark.md` §4 |
| NFR-003, NFR-006 | Benchmark vetorial com *ground truth* exato | `./Benchmark.md` §5 |
| NFR-005 | Uptime probe + *error budget* mensal | `./Monitoring.md` §3 |
| NFR-007 | Teste de escala (10⁹ linhas particionadas) | `./Scalability.md` §7 |
| NFR-008 | Chaos test (kill do primário sob carga) | `./FailureRecovery.md` §5 |
| NFR-009 | DR drill trimestral + `restore drill` semanal | `./FailureRecovery.md` §6 |
| NFR-010 | Medição de `aios_db_migration_block_ms` em CI e produção | `./Testing.md` §5 |
| NFR-011 | Auditoria contínua + pentest de RLS | `./Security.md` §6 |
| NFR-012, NFR-013 | Teste de concorrência e de lag | `./Testing.md` §7 |
| NFR-014 | Inspeção de cobertura de spans e campos de log | `./Logging.md` §4 |
| NFR-015 | Teste de replay com `Idempotency-Key` | `./Testing.md` §4 |
| NFR-016 | Coleta de `aios_db_table_bloat_ratio` | `./Metrics.md` |

---

## 5. Requisitos Herdados (não redefinidos aqui)

| Origem | Requisito herdado | Aplicação no módulo |
|--------|-------------------|---------------------|
| RFC-0001 §5.1 | Identidade de recurso em URN com ULID | PK `urn` em `platform.*` e nas tabelas de domínio |
| RFC-0001 §5.2–§5.3 | Envelope CloudEvents e convenção de subjects | `./Events.md` |
| RFC-0001 §5.4 | Envelope de erro `AIOS-<DOMINIO>-<NNNN>` | Domínios `DB` e `MIGRATION` em `./API.md` |
| RFC-0001 §5.5 | Idempotência por `Idempotency-Key` (≥24h) | Toda mutação administrativa |
| RFC-0001 §5.6 | Cabeçalhos de correlação | `./Logging.md`, `./API.md` |
| RFC-0001 §5.8 | PEP/PDP com *default deny*; auditoria de ação privilegiada | FR-017 |
| RFC-0001 §6–§7 | mTLS interno, minimização e retenção de dados pessoais | `./Security.md` |
| `../000-Vision/Vision.md` V-R10 | RTO ≤ 15 min, RPO ≤ 5 min | NFR-009 |

---

## 6. Glossário Local

Este módulo **não define termos próprios**. Todos os termos usados
(*Tenant*, *Shard*, *Control Plane*, *Data Plane*, *PEP*, *PDP*, *Outbox*) são os do
`../040-Glossary/Glossary.md`. Termos específicos de PostgreSQL (partição, WAL, LSN,
RLS, HNSW, IVFFlat, advisory lock) são usados com o significado padrão da
documentação oficial do PostgreSQL 16, do `pgvector` e do Apache AGE, e estão
contextualizados em `./Database.md`.

---

## 7. Referências

- Requisitos funcionais: `./FunctionalRequirements.md`
- Requisitos não-funcionais: `./NonFunctionalRequirements.md`
- Casos de uso: `./UseCases.md` · Testes: `./Testing.md`
- Brief: `./_DESIGN_BRIEF.md` · Contratos: `../003-RFC/RFC-0001-Architecture-Baseline.md`
