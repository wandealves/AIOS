---
Documento: Architecture
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0001, ADR-0005, ADR-0050, ADR-0051, ADR-0052, ADR-0053, ADR-0054, ADR-0057, ADR-0058, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: 001-Architecture, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit, 027-Cluster, 028-Deployment
---

# 005-Database — Arquitetura do Módulo

Este documento deriva de `./_DESIGN_BRIEF.md` §0, §2, §3 e §10. Onde houver
divergência, o brief prevalece e este documento é o defeito.

---

## 1. Visão C4 — Nível 1: Contexto

```
   ┌────────────────┐     ┌────────────────┐     ┌────────────────┐
   │ 030-CLI ·      │     │ Engenheiro de  │     │ SRE / Operador │
   │ 032-WebConsole │     │ módulo         │     │ de plataforma  │
   └───────┬────────┘     └───────┬────────┘     └───────┬────────┘
           │ (admin)              │ (migração)           │ (DR, runbooks)
           └──────────────┬───────┴──────────────────────┘
                          ▼
                  ┌───────────────┐
                  │   004-API     │  (única porta externa; AuthN OIDC)
                  └───────┬───────┘
                          ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                005-DATABASE — Plataforma de Dados                     │
   │  API administrativa (aios.database.v1) + operação contínua do cluster │
   └───┬──────────┬───────────┬───────────┬───────────┬───────────┬───────┘
       │          │           │           │           │           │
       ▼          ▼           ▼           ▼           ▼           ▼
   022-Policy  025-Audit  024-Observ.  027-Cluster  021-Security  020-Comm.
    (PDP)      (trilha)   (OTel)       (failover)   (segredos)    (NATS)

   ┌──────────────────────────────────────────────────────────────────────┐
   │ CONSUMIDORES DE PERSISTÊNCIA (acesso SQL direto, via PgBouncer)      │
   │ 006 Kernel · 007 Runtime · 008 Lifecycle · 009 Scheduler · 010 Memory│
   │ 011 Context · 012–019 · 022 Policy · 023 Learning · 025 Audit · 026  │
   └──────────────────────────────────────────────────────────────────────┘
```

**Leitura:** há **dois caminhos** distintos para o módulo. O caminho
**administrativo** (migrar, expurgar, restaurar) passa pelo `004-API` e é sempre
autorizado pelo PDP. O caminho de **dados** (o `SELECT`/`INSERT` de cada módulo)
não passa pelo serviço 005: vai direto ao PostgreSQL através do
`ConnectionPoolGateway`, sob as *roles*, RLS e limites que este módulo governa. Essa
separação é deliberada — colocar o serviço 005 no caminho quente de dados criaria um
salto de rede desnecessário e um ponto único de falha para toda escrita do sistema.

---

## 2. Visão C4 — Nível 2: Contêineres

| Contêiner | Tecnologia | Papel | Réplicas |
|-----------|-----------|-------|----------|
| `aios-database-svc` | .NET 10 | Serviço de plataforma: migração, catálogo, retenção, backup, replicação, API administrativa. **Stateless**. | 2 (HA ativo/passivo por *advisory lock*) |
| `aios-postgres-primary` | PostgreSQL 16 + pgvector + AGE | Fonte da verdade transacional; único destino de escrita. | 1 (com failover) |
| `aios-postgres-replica` | PostgreSQL 16 (streaming) | Leituras elegíveis, *dry-run* de migração, candidato a promoção. | ≥ 2 |
| `aios-pgbouncer` | PgBouncer (modo `transaction`) | Fronteira de conexão, bulkhead por serviço, roteamento leitura/escrita. | 2 |
| `aios-wal-archiver` | *sidecar* de arquivamento | Envio contínuo de WAL a MinIO (limita o RPO). | 1 por primário |
| MinIO | S3-compatível | Destino de WAL e backups (full/incremental). | externo (`028`) |
| NATS/JetStream | NATS | Publicação dos eventos `aios._platform.database.*`. | externo (`020`) |

```
        ┌──────────────────────┐            ┌────────────────────────┐
        │  aios-database-svc   │──gRPC/REST─│  004-API (borda)       │
        │  (.NET 10, stateless)│            └────────────────────────┘
        └───┬───────────┬──────┘
    DDL/DML │           │ eventos (Outbox → JetStream)
            ▼           ▼
   ┌───────────────┐  NATS
   │ aios-pgbouncer│◀───────────── módulos consumidores (006…026)
   └───┬───────┬───┘
       │       └──────────── leituras elegíveis ─────────┐
       ▼                                                  ▼
  ┌───────────────┐   streaming replication   ┌────────────────────┐
  │ postgres-     │──────────────────────────▶│ postgres-replica ×N│
  │ primary       │                            └────────────────────┘
  └───────┬───────┘
          │ WAL contínuo + backup full/incremental
          ▼
     ┌─────────┐
     │  MinIO  │  (janela de PITR)
     └─────────┘
```

---

## 3. Visão C4 — Nível 3: Componentes

Os componentes internos e suas responsabilidades são normativos no
`./_DESIGN_BRIEF.md` §2.1. Resumo com a fronteira de cada um:

| Componente | Responsabilidade sintética | Fronteira (o que NÃO faz) |
|------------|---------------------------|---------------------------|
| `SchemaRegistry` | Catálogo de schemas/tabelas/donos/classificação. | Não interpreta a semântica do dado. |
| `MigrationEngine` | Executa a FSM de migração sob *advisory lock*. | Não escreve DDL — recebe do módulo dono. |
| `DdlConventionValidator` | Lint de DDL (convenções, DDL perigoso, bloqueio). | Não avalia mérito de modelagem de domínio. |
| `RlsPolicyManager` | Gera/aplica/audita RLS e *roles* por serviço. | Não define quem pode o quê no domínio (é do `022`). |
| `PartitionManager` | Pré-cria e remove partições conforme política. | Não decide TTL (é do `RetentionEnforcer`). |
| `RetentionEnforcer` | Aplica TTL, preferindo *drop de partição*. | Não apaga sob `legal_hold`. |
| `ErasureCoordinator` | Direito ao esquecimento com comprovante. | Não decide legalidade — consome `legal_basis`. |
| `VectorIndexManager` | Governa índices `pgvector` e recall. | Não escolhe modelo de embedding (`010`/`017`/`018`). |
| `GraphStoreManager` | Governa grafos AGE e limites de travessia. | Não define ontologia (`019`). |
| `BackupOrchestrator` | Backup, WAL archiving, catálogo, *restore drill*. | Não decide topologia de DR (`027`). |
| `ReplicationSupervisor` | Lag, elegibilidade de leitura, sinal de promoção. | Não elege líder (`027`). |
| `ConnectionPoolGateway` | Pooling, bulkhead, timeouts, roteamento R/W. | Não reescreve query. |
| `QueryGovernor` | Captura de plano, corte de query patológica. | Não cria índices sozinho — recomenda. |
| `PolicyClient` | PEP → PDP com CB, cache curto, *fail-closed*. | Não decide (é PEP). |
| `EventEmitter` | Outbox transacional → JetStream. | Não define subjects próprios fora da RFC-0001. |
| `DbTelemetry` | Spans/métricas/logs OTel + auditoria. | Não armazena a trilha (é do `025`). |

O diagrama completo de componentes é o do `./_DESIGN_BRIEF.md` §2.2 e é reproduzido
em `./ClassDiagrams.md` §2 com as interfaces.

---

## 4. Padrões Arquiteturais Adotados

| Padrão | Onde | Por quê | ADR |
|--------|------|---------|-----|
| **Schema por módulo dono** | Organização do banco | Fronteira de propriedade explícita e privilégio mínimo por *role*; evita banco/schema por tenant (que multiplicaria objetos por 10³–10⁴ tenants). | ADR-0050 |
| **Row-Level Security** | Toda tabela multi-tenant | Isolamento independente do código de aplicação; um defeito de query vira zero linhas, não vazamento. | ADR-0051 |
| **Convenções impostas por linter** | Pipeline de migração | Consistência física (URN, `tenant_id`, `version`, `timestamptz`) verificável antes do deploy. | ADR-0052 |
| **Expand/Contract + forward-only** | Migrações | Permite coexistência de versões de aplicação durante rollout (RFC-0001 §5.7); elimina rollback destrutivo. | ADR-0053 |
| **Advisory lock global** | Aplicação de migração | Garante uma migração por vez em todo o cluster, sem depender de disciplina operacional. | ADR-0053 |
| **Particionamento declarativo** | Séries e alta cardinalidade | RANGE temporal para séries; HASH por `tenant_id` para distribuir *hot tenants*. | ADR-0054 |
| **Retenção por drop de partição** | Expurgo por TTL | Custo O(1) e ausência de *bloat*, ao contrário de `DELETE` em massa. | ADR-0055 |
| **WAL archiving contínuo + PITR** | Durabilidade | Recuperação a qualquer LSN dentro da janela; RPO limitado pelo `wal_archive_timeout_s`. | ADR-0056 |
| **Outbox transacional** | Emissão de eventos | Atomicidade entre mudança de estado e publicação; dedupe por `event.id` (RFC-0001 §5.5). | herdado de ADR-0006 |
| **PEP → PDP** | Operações administrativas | *Default deny* em toda ação privilegiada (RFC-0001 §5.8). | herdado de ADR-0008 |
| **Bulkhead de conexões** | PgBouncer | Um serviço não drena o pool dos demais; contenção de *noisy neighbor*. | ADR-0059 |

---

## 5. Decisões Tecnológicas e Alternativas Descartadas

| Escolha | Alternativas consideradas | Por que a escolha | Trade-off aceito |
|---------|---------------------------|-------------------|------------------|
| **PostgreSQL 16** como fonte da verdade | MySQL; CockroachDB; Cassandra | DDL transacional (viabiliza I2 da FSM), RLS nativa, particionamento declarativo, ecossistema de extensões. | Escala horizontal de escrita exige sharding aplicativo, não é automática. |
| **pgvector** no mesmo cluster | Qdrant/Milvus/Weaviate dedicados | Uma transação cobre dado + vetor: sem consistência eventual entre "o item" e "o embedding do item"; menos um sistema para operar. | Menor teto de escala vetorial que um motor dedicado; mitigado por HNSW e particionamento. |
| **Apache AGE** no mesmo cluster | Neo4j dedicado | Mesma justificativa: grafo do `019-GraphRAG` transacionalmente coerente com as entidades; RLS reaproveitada. | Menor maturidade de algoritmos de grafo; travessia limitada por profundidade. |
| **RLS** para multi-tenancy | Schema por tenant; banco por tenant | Um schema com 10⁴ tenants é operável; 10⁴ schemas não são (migração vira O(n) e o catálogo do PG degrada). | Toda sessão precisa fixar o `tenant_id`; custo de planejamento levemente maior. |
| **PgBouncer em modo `transaction`** | Pool na aplicação; modo `session` | Multiplexa milhares de conexões lógicas em dezenas físicas — condição para 10⁶ agentes. | Proíbe *prepared statements* de sessão e `SET` persistente; codificado como regra do linter. |
| **Forward-only** | Migrações reversíveis por padrão | Rollback de DDL destrutivo é ilusório em produção com dado novo já gravado. | Exige disciplina de expand/contract e mais migrações pequenas. |

Cada linha acima DEVE ser ratificada pela ADR correspondente em `./ADR.md`; nenhuma
decisão é tomada localmente neste documento.

---

## 6. Fluxo Arquitetural Principal — Aplicação de Migração

```
 Módulo dono        004-API      MigrationEngine   DdlValidator   PDP(022)   PG primário
     │                 │               │                │            │           │
     │ POST migrations │               │                │            │           │
     ├────────────────▶│──── gRPC ────▶│  T-01 Draft    │            │           │
     │                 │               ├───lint────────▶│            │           │
     │                 │               │◀──ok/violação──┤            │           │
     │                 │               │  T-02 Validated│            │           │
     │                 │               ├── dry-run em réplica ───────────────────▶│(réplica)
     │                 │               │  T-04 DryRun   │            │           │
     │  POST :apply    │               │                │            │           │
     ├────────────────▶│──────────────▶│── capability? ─────────────▶│           │
     │                 │               │◀──── allow ────────────────┤            │
     │                 │               ├── pg_advisory_lock ────────────────────▶│
     │                 │               │  T-06 Applying │            │           │
     │                 │               ├── BEGIN; DDL; UPDATE registry; INSERT outbox; COMMIT ▶│
     │                 │               │  T-07 Applied  │            │           │
     │◀── 200 + URN ───┤◀──────────────┤                │            │           │
                                        └─ relay publica aios._platform.database.migration.applied
```

Os fluxos de falha (lint reprovado, lock ocupado, timeout na aplicação) estão em
`./SequenceDiagrams.md` §4–§6.

---

## 7. Fronteiras de Confiança

```
   ┌─── Zona não confiável ───────────────────────────────────────────┐
   │ CLI · WebConsole · integrações externas                          │
   └──────────────────────────┬───────────────────────────────────────┘
                              │ TLS + OIDC (validação no 004-API)
   ┌──────────────────────────▼───────────────────────────────────────┐
   │ Zona de serviços (malha interna, mTLS obrigatório)               │
   │  aios-database-svc  ·  módulos consumidores                      │
   └──────────────────────────┬───────────────────────────────────────┘
                              │ TLS + credencial por *role* (cofre 021)
   ┌──────────────────────────▼───────────────────────────────────────┐
   │ Zona de dados: PostgreSQL (RLS ativa) · réplicas · MinIO cifrado │
   │  ← última linha de defesa: mesmo com bug de aplicação, RLS isola │
   └──────────────────────────────────────────────────────────────────┘
```

Detalhamento em `./Security.md`.

---

## 8. Atributos de Qualidade Arquiteturalmente Significativos

| Atributo | Decisão arquitetural que o sustenta | NFR |
|----------|-------------------------------------|-----|
| Latência estável sob volume | Particionamento + retenção por *drop* mantêm o índice ativo pequeno. | NFR-001, NFR-007 |
| Isolamento | RLS + *roles* por serviço + auditoria contínua. | NFR-011 |
| Durabilidade | `synchronous_commit=on` + réplica síncrona no quórum + WAL archiving. | NFR-008, NFR-009 |
| Evolutibilidade | Expand/contract, lint, *dry-run*, orçamento de bloqueio de 200 ms. | NFR-010 |
| Contenção de falhas | Bulkhead de conexões, `statement_timeout`, limite de travessia AGE. | NFR-004, NFR-016 |
| Observabilidade | OTel em toda operação administrativa; métricas `aios_db_*`. | NFR-014 |

---

## 9. Riscos Arquiteturais

| Risco | Impacto | Mitigação |
|-------|---------|-----------|
| Primário PostgreSQL como gargalo de escrita | Teto de throughput global. | Sharding lógico por `shard_key` prepara a divisão em clusters por faixa (`../027-Cluster/`); réplicas absorvem leitura. |
| RLS desabilitada por engano em migração | Vazamento entre tenants. | Lint rejeita DDL que desabilite RLS; auditoria contínua (`schemas:audit`) emite `compliance.violated` como P1. |
| Reindexação vetorial custosa | Janela de degradação em `010`/`018`. | `REINDEX CONCURRENTLY`, dimensão imutável por coluna, mudança de parâmetro exige migração planejada. |
| Crescimento de custo de armazenamento | Custo por tenant fora de controle. | `bytes_estimate` por tabela, cota por tenant (`AIOS-DB-0012`), integração com `../026-Cost-Optimizer/`. |
| Modo `transaction` do PgBouncer quebra padrões de acesso | Erros sutis em runtime. | Regra de lint + `Testing.md` §6 (teste de contrato de acesso) + documentação em `./FAQ.md`. |

---

## 10. Referências

- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Brief do módulo: `./_DESIGN_BRIEF.md`
- Modelo físico: `./Database.md` · API: `./API.md` · Eventos: `./Events.md`
- Escala: `./Scalability.md` · Falhas: `./FailureRecovery.md` · Segurança: `./Security.md`
- Decisões: `./ADR.md` · Especificações: `./RFC.md`
