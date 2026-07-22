---
Documento: ADR
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0001, ADR-0002, ADR-0005, ADR-0006, ADR-0008, ADR-0010; ADR-0050..ADR-0059
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: 002-ADR, _DESIGN_BRIEF.md §11
---

# 005-Database — Índice de ADRs

Faixa reservada a este módulo: **`ADR-0050`..`ADR-0059`** (regra `NNN × 10`). As ADRs
são registradas em `../002-ADR/` — este documento é apenas o índice com o impacto de
cada decisão sobre o módulo.

---

## 1. ADRs Globais Herdadas

| ADR | Decisão | Impacto no 005-Database | Status |
|-----|---------|-------------------------|--------|
| ADR-0001 | Arquitetura em plano de controle (.NET) e plano de dados (Python) | O `aios-database-svc` é serviço do plano de controle. | Accepted |
| ADR-0002 | Contratos REST + gRPC versionados | API administrativa em `/v1/database` e `aios.database.v1`. | Accepted |
| **ADR-0005** | **PostgreSQL + pgvector + Apache AGE como plataforma de dados** | **Decisão fundadora deste módulo**: define o motor, as extensões e o modelo relacional como base de todo o AIOS. | Accepted |
| ADR-0006 | NATS/JetStream como barramento; padrão Outbox | Define o contrato da tabela `outbox` e a emissão transacional de eventos. | Accepted |
| ADR-0008 | PEP/PDP com *default deny* | Toda operação administrativa consulta o PDP do `../022-Policy/`. | Accepted |
| ADR-0010 | Observabilidade OTel obrigatória | Métricas `aios_db_*`, traces e logs correlacionados. | Accepted |

---

## 2. ADRs a Propor por Este Módulo

| ADR | Título | Escopo da decisão | Documentos afetados | Status |
|-----|--------|-------------------|---------------------|--------|
| ADR-0050 | Schema por módulo dono, catálogo central `platform.schema_registry` e registro do domínio de eventos `database` | Organização do banco por propriedade; criação do domínio `database` no registro da RFC-0001 §8; uso do tenant `_platform`. | `Architecture.md`, `Database.md`, `Events.md` | Proposed |
| ADR-0051 | RLS por `tenant_id` como fronteira física de isolamento | RLS vs. schema por tenant vs. banco por tenant. | `Security.md`, `Database.md`, `Scalability.md` | Proposed |
| ADR-0052 | Convenções físicas obrigatórias e o *linter* de DDL que as impõe | URN como PK, `tenant_id`, `version` (OCC), `timestamptz`, índice em FK; regras CV-01..CV-12. | `Database.md`, `ClassDiagrams.md`, `Testing.md` | Proposed |
| ADR-0053 | Migrações forward-only com expand/contract e *advisory lock* global | Direção da evolução, exclusão mútua, orçamento de bloqueio de 200 ms. | `StateMachine.md`, `Deployment.md`, `Testing.md` | Proposed |
| ADR-0054 | Estratégia de particionamento: RANGE temporal e HASH por `tenant_id` | Quando usar cada estratégia; `hash_modulus` como decisão de longo prazo. | `Database.md`, `Scalability.md` | Proposed |
| ADR-0055 | Retenção por *drop de partição* como mecanismo primário de expurgo | `DROP` vs. `DELETE` em lote; consequências para *bloat* e custo. | `Database.md`, `FailureRecovery.md`, `Metrics.md` | Proposed |
| ADR-0056 | Backup contínuo (WAL archiving em MinIO) + PITR com *restore drill* como critério de validade | Backup não verificado é backup inexistente. | `FailureRecovery.md`, `Deployment.md`, `Monitoring.md` | Proposed |
| ADR-0057 | pgvector: HNSW como índice default, dimensão imutável por coluna, reindexação online | HNSW vs. IVFFlat; parâmetros default (`m=16`, `ef_construction=64`, `ef_search=64`). | `Database.md`, `Benchmark.md` | Proposed |
| ADR-0058 | Apache AGE para o grafo de conhecimento no mesmo cluster | AGE vs. banco de grafos dedicado (Neo4j); coerência transacional vs. maturidade de algoritmos. | `Database.md`, `Architecture.md` | Proposed |
| ADR-0059 | PgBouncer em modo `transaction` com bulkhead por serviço; domínios de erro `DB` e `MIGRATION` | Pooling, limites de conexão, restrições de padrão de acesso; reserva dos domínios de erro. | `API.md`, `Configuration.md`, `Scalability.md` | Proposed |

---

## 3. Decisões e Alternativas Descartadas (resumo)

| ADR | Escolha | Alternativas descartadas | Trade-off aceito |
|-----|---------|--------------------------|------------------|
| ADR-0051 | RLS | Schema por tenant; banco por tenant | Toda sessão precisa fixar `tenant_id`; custo de planejamento levemente maior. |
| ADR-0053 | Forward-only | Migrações reversíveis por padrão | Exige mais migrações pequenas e disciplina de expand/contract. |
| ADR-0055 | `DROP` de partição | `DELETE` em lote com autovacuum | Exige particionar desde o início; retenção não-temporal continua usando `delete_batch`. |
| ADR-0057 | HNSW | IVFFlat; motor vetorial dedicado | Índice maior em RAM; teto de escala inferior a um motor especializado. |
| ADR-0058 | AGE no mesmo cluster | Neo4j dedicado | Menos algoritmos de grafo prontos; travessia limitada por profundidade. |
| ADR-0059 | PgBouncer `transaction` | Pool na aplicação; modo `session` | Proíbe estado de sessão (prepared statement nomeado, `SET` persistente, `LISTEN/NOTIFY`). |

---

## 4. Processo

1. Uma decisão arquitetural do módulo **NÃO DEVE** ser tomada dentro de um dos 26
   documentos: ela é registrada como ADR em `../002-ADR/` e apenas **referenciada** aqui.
2. Novas ADRs deste módulo usam a próxima numeração livre dentro de
   `ADR-0050`..`ADR-0059`. Esgotada a faixa, a extensão é decidida com a arquitetura-chefe.
3. Uma ADR que altere contrato central exige também **RFC** (ver `./RFC.md`).
4. Ao mudar o status de uma ADR (`Proposed` → `Accepted` / `Superseded`), atualize esta
   tabela e o índice em `../002-ADR/README.md`.

---

## 5. Referências

- Índice global de decisões: `../002-ADR/README.md`
- Brief (origem das propostas): `./_DESIGN_BRIEF.md` §11
- RFCs do módulo: `./RFC.md`
