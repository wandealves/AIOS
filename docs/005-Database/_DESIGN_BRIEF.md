---
Documento: Design Brief Interno (Fonte Única de Verdade do Módulo)
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0001, ADR-0002, ADR-0005, ADR-0006, ADR-0008, ADR-0010 (globais, herdados); ADR-0050..ADR-0059 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline, consumida — não redefinida); RFC-0050 (Data Platform Contract), RFC-0051 (Migration Protocol), a propor por este módulo
Depende de: 001-Architecture, 003-RFC (RFC-0001), 020-Communication (NATS/JetStream), 021-Security (segredos, mTLS, identidades de banco), 022-Policy (PDP), 024-Observability, 025-Audit, 027-Cluster (HA/DR/failover), 028-Deployment (topologia), 029-Operations (runbooks); consumido como plataforma de persistência por 006, 007, 008, 009, 010, 011, 012, 013, 014, 015, 016, 017, 018, 019, 022, 023, 025, 026
---

# 005-Database — Design Brief (Fonte Única de Verdade)

> **Escopo deste documento.** Este brief é a fonte única de verdade **interna** do
> módulo 005-Database. Os 26 documentos obrigatórios (ver `README.md` e
> `../_templates/MODULE_TEMPLATE.md`) DEVEM ser derivados deste brief e **NÃO PODEM
> contradizê-lo**. Onde um contrato central já existe (URN, envelope de evento,
> envelope de erro, idempotência, correlação, subjects, versionamento), este
> documento **referencia** a `../003-RFC/RFC-0001-Architecture-Baseline.md` e
> **não o redefine**. Termos são os do `../040-Glossary/Glossary.md`.

> **Palavras normativas** conforme RFC 2119 / RFC 8174: DEVE, NÃO DEVE, DEVERIA,
> NÃO DEVERIA, PODE.

---

## 0. Posição no AIOS e contrato de fronteira

```
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  MÓDULOS DONOS DE DOMÍNIO (006 Kernel · 009 Scheduler · 010 Memory ·      │
   │  011 Context · 015 Tool · 018 Knowledge · 019 GraphRAG · 022 · 025 · …)   │
   │  · definem o CONTEÚDO lógico do seu schema (tabelas, colunas, semântica)  │
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ DDL versionado (migração) + queries em runtime
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  005-DATABASE — PLATAFORMA DE DADOS (este módulo)                         │
   │  · convenções físicas obrigatórias (URN/ULID, tenant_id, RLS, OCC)        │
   │  · pipeline de migração governado (expand/contract, forward-only)         │
   │  · particionamento, retenção, expurgo (LGPD), backup/PITR, replicação     │
   │  · governança de extensões: pgvector (010/011/018) e Apache AGE (019)     │
   └───────────────┬──────────────────────────────────────────────────────────┘
                   │ instancia e opera
                   ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  PostgreSQL 16 (primário + réplicas) · pgvector · Apache AGE · PgBouncer  │
   │  WAL archiving → MinIO   (Redis e MinIO NÃO pertencem a este módulo)      │
   └──────────────────────────────────────────────────────────────────────────┘
```

**Fronteira em uma frase:** *o módulo dono decide **o que** persistir; o 005 decide
**como** isso é fisicamente armazenado, isolado, migrado, particionado, retido,
recuperado e observado.*

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Missão

O 005-Database é o **subsistema de armazenamento** do AIOS — o análogo do
*filesystem + volume manager* de um sistema operacional clássico. Ele não conhece a
semântica cognitiva de um ACB, de um item de memória ou de um plano; conhece
**páginas, esquemas, índices, partições, transações, isolamento e durabilidade**.
Sua missão é oferecer a todos os módulos do plano de controle uma **plataforma de
persistência única, multi-tenant por construção e evolutiva sem downtime**.

A fronteira de confiança do módulo é o **isolamento multi-tenant no nível físico**:
mesmo que um módulo consumidor tenha um defeito de query, a **Row-Level Security
(RLS)** por `tenant_id` DEVE impedir vazamento entre tenants. Esse é o invariante
mais forte do módulo (ver §12) e a razão pela qual a criação de tabelas é um ato
governado, não livre: nenhuma tabela multi-tenant entra em produção sem `tenant_id`,
sem política RLS e sem política de retenção declarada.

O módulo também é o **guardião da evolução do esquema**: toda mudança de DDL é uma
**migração versionada, forward-only e revisada**, aplicada pelo padrão
**expand/contract** para que schema antigo e novo coexistam durante o rollout
(alinhado a `../028-Deployment/` e à regra de coexistência de versões da RFC-0001 §5.7).

### 1.2 Responsabilidades (o 005-Database DEVE)

| # | Responsabilidade |
|---|------------------|
| R-01 | Manter o **modelo físico consolidado** do AIOS: um schema PostgreSQL por módulo dono (`kernel`, `sched`, `memory`, `context`, `tools`, `knowledge`, `graph`, `policy`, `audit`, `cost`, `platform`), com catálogo de propriedade em `platform.schema_registry`. |
| R-02 | Definir e **fazer cumprir as convenções físicas obrigatórias**: chave primária `urn` (RFC-0001 §5.1), `tenant_id` em toda tabela multi-tenant, `version bigint` para OCC, `created_at`/`updated_at` `timestamptz` em UTC, nomes em `snake_case`. |
| R-03 | Gerar, validar e ativar as **políticas de Row-Level Security (RLS)** por `tenant_id` e as *roles* de banco por serviço (privilégio mínimo, sem `SUPERUSER` em runtime). |
| R-04 | Operar o **pipeline de migração governado** (`platform.migration`): submissão, validação estática (lint de convenções), *dry-run* em réplica, aplicação forward-only com *advisory lock* global, e registro imutável do resultado. |
| R-05 | Gerenciar **particionamento** (por tempo para tabelas de série temporal; por `hash(tenant_id)` para tabelas de alta cardinalidade), incluindo **pré-criação** de partições futuras e *detach/drop* das expiradas. |
| R-06 | Aplicar **políticas de retenção** por tabela (`platform.retention_policy`) e executar o **expurgo rastreável** que materializa o direito ao esquecimento (LGPD/GDPR), coordenado com `025-Audit`. |
| R-07 | Garantir **durabilidade e recuperabilidade**: replicação em streaming, *WAL archiving* contínuo para MinIO, backups full+incrementais, **PITR** e verificação periódica de restauração (*restore drill* automatizado). |
| R-08 | Governar as **extensões**: `pgvector` (tipos, dimensões, índices HNSW/IVFFlat, operadores de distância) e **Apache AGE** (grafos, *labels*, isolamento por tenant), incluindo versão e reindexação. |
| R-09 | Governar o **acesso e a concorrência**: pooling (PgBouncer), cota de conexões por serviço, `statement_timeout`/`idle_in_transaction_session_timeout`, e detecção de *slow query*/plano regredido. |
| R-10 | Publicar o **contrato canônico da tabela `outbox`** (padrão Outbox transacional usado por 006, 009, 010 e demais) e garantir suas propriedades físicas (índice de varredura, retenção, particionamento). |
| R-11 | Atuar como **PEP**: toda operação administrativa (aplicar migração, expurgar dados, restaurar backup, alterar retenção) consulta o **PDP** do `022-Policy` antes de executar. *Default deny*. |
| R-12 | Emitir **eventos** de plataforma (migração, partição, retenção, backup, replicação) no NATS com o envelope CloudEvents da RFC-0001 §5.2, e **telemetria OTel** (`aios_db_*`) + auditoria via `025-Audit`. |

### 1.3 Não-Responsabilidades (o 005-Database NÃO DEVE)

| # | Não-Responsabilidade | Dono real |
|---|----------------------|-----------|
| NR-01 | Definir a **semântica** das entidades de domínio (o que é um ACB, um item de memória, um plano). O 005 hospeda; o dono define. | `006`, `010`, `012`, … (módulo dono do schema) |
| NR-02 | Executar lógica de negócio, orquestração ou regra cognitiva dentro do banco (sem *stored procedures* de domínio). | Módulos de domínio |
| NR-03 | **Decidir** política de autorização (ser PDP). O módulo é PEP. | `022-Policy` |
| NR-04 | Persistir a **trilha de auditoria imutável** (o 005 fornece o schema e a retenção; a semântica, o encadeamento e a imutabilidade lógica são do Audit). | `025-Audit` |
| NR-05 | Gerenciar o **estado quente volátil** (contadores de cota, cache de decisão, ACB *hot*) em **Redis**. | `006-Kernel`, `009-Scheduler` (uso), `028-Deployment` (provisão) |
| NR-06 | Gerenciar **objetos binários** (checkpoints, snapshots, artefatos) em **MinIO** — exceto os arquivos de WAL/backup do próprio PostgreSQL. | `008-Agent-Lifecycle`, `028-Deployment` |
| NR-07 | Decidir a **estratégia de recuperação/failover do cluster** como um todo (quórum, eleição de líder, DR entre regiões). O 005 executa a parte de dados. | `027-Cluster` |
| NR-08 | Emitir/validar tokens, gerenciar identidade federada ou custodiar segredos (o 005 **consome** credenciais do cofre). | `021-Security` |
| NR-09 | Definir a **estratégia de embedding** (modelo, dimensionalidade semântica, *chunking*). O 005 provê o índice vetorial e o operador de distância. | `010-Memory`, `018-Knowledge`, `017-Model-Router` |
| NR-10 | Definir o **modelo do grafo de conhecimento** (ontologia, *labels*, algoritmos de travessia). O 005 provê o AGE governado. | `019-GraphRAG` |
| NR-11 | Definir **orçamento de custo** de armazenamento por tenant (o 005 mede e aplica limites; o orçamento é do 026). | `026-Cost-Optimizer` |
| NR-12 | Expor consultas de domínio a clientes externos (não há acesso direto ao banco a partir da borda; todo acesso externo passa pelo `004-API`). | `004-API` |

---

## 2. Decomposição em Componentes Internos

### 2.1 Componentes (PascalCase · responsabilidade · dependências)

| Componente | Responsabilidade | Depende de (interno → externo) |
|------------|------------------|-------------------------------|
| **SchemaRegistry** | Catálogo autoritativo de schemas, tabelas, donos (módulo), classificação de dados (PII/sensível) e versão de schema aplicada. Fonte da verdade de "quem é dono do quê". | → PostgreSQL (`platform.*`) |
| **MigrationEngine** | Executa a FSM de migração (§4): valida, faz *dry-run*, aplica forward-only sob *advisory lock* global, registra resultado, dispara compensação em falha. | SchemaRegistry, DdlConventionValidator, EventEmitter; → PostgreSQL |
| **DdlConventionValidator** | *Linter* de DDL: rejeita tabela multi-tenant sem `tenant_id`/RLS/retenção, PK fora do padrão URN, índice ausente em FK, tipo `timestamp` sem *timezone*, DDL bloqueante fora da janela permitida. | SchemaRegistry |
| **RlsPolicyManager** | Gera, aplica e audita políticas RLS por `tenant_id`; cria *roles* por serviço com privilégio mínimo; verifica continuamente que nenhuma tabela multi-tenant está com RLS desabilitada. | SchemaRegistry; → PostgreSQL |
| **PartitionManager** | Cria partições futuras com antecedência, faz *detach/drop* das expiradas, escolhe estratégia (RANGE por tempo, HASH por `tenant_id`) e mantém `platform.partition_policy`. | SchemaRegistry, RetentionEnforcer; → PostgreSQL |
| **RetentionEnforcer** | Aplica `platform.retention_policy` (TTL por tabela), preferindo *drop de partição* a `DELETE`; registra volume expurgado; nunca apaga o que está sob *legal hold*. | PartitionManager, EventEmitter; → 025-Audit |
| **ErasureCoordinator** | Executa o direito ao esquecimento: recebe pedido, resolve o grafo de tabelas que referenciam o titular, aplica *delete*/tokenização em ordem segura de FK, e emite comprovante rastreável. | SchemaRegistry, RetentionEnforcer, PolicyClient; → 025-Audit |
| **VectorIndexManager** | Governa `pgvector`: dimensões declaradas por coluna, escolha e parâmetros de índice (HNSW `m`/`ef_construction`; IVFFlat `lists`), reindexação online e medição de *recall*. | SchemaRegistry; → PostgreSQL/pgvector |
| **GraphStoreManager** | Governa **Apache AGE**: criação de grafos por tenant, *labels* permitidos, limites de travessia e índices de propriedade; isolamento equivalente ao RLS para vértices/arestas. | SchemaRegistry, RlsPolicyManager; → PostgreSQL/AGE |
| **BackupOrchestrator** | Agenda backup full/incremental, arquiva WAL continuamente em MinIO, mantém `platform.backup_catalog`, executa *restore drill* periódico e calcula RPO observado. | EventEmitter; → MinIO, PostgreSQL |
| **ReplicationSupervisor** | Supervisiona réplicas de streaming: mede *lag*, roteia leituras elegíveis para réplica, sinaliza `027-Cluster` para promoção e bloqueia leitura obsoleta acima do limiar. | ConnectionPoolGateway, EventEmitter; → 027-Cluster |
| **ConnectionPoolGateway** | Fronteira de conexão: PgBouncer em modo *transaction*, cota de conexões e *bulkhead* por serviço, roteamento leitura/escrita, aplicação de `statement_timeout`. | ReplicationSupervisor; → PgBouncer/PostgreSQL |
| **QueryGovernor** | Observa e contém consultas patológicas: captura de plano, detecção de regressão, *kill* de query acima do orçamento, e relatório de índices ausentes por schema. | DbTelemetry; → PostgreSQL (`pg_stat_statements`) |
| **PolicyClient** | Cliente resiliente do PDP (`022-Policy`) para toda operação administrativa; circuit breaker, cache curto, *fail-closed*. | → 022-Policy |
| **EventEmitter** | Publica eventos CloudEvents no NATS/JetStream via **Outbox transacional** em `platform.outbox`; garante at-least-once e dedupe por `event.id`. | → PostgreSQL, NATS/JetStream |
| **DbTelemetry** | Instrumentação OTel transversal: spans de migração/expurgo/restauração, métricas `aios_db_*`, logs Serilog correlacionados e emissão de auditoria para `025`. | → 024-Observability, 025-Audit |

### 2.2 Diagrama de Componentes (ASCII)

```
                REST (admin, via 004-API) / gRPC (interno, aios.database.v1)
                                          │
   ┌──────────────────────────────────────▼──────────────────────────────────────┐
   │                     DATABASE PLATFORM SERVICE (005 · .NET 10)                │
   │                                                                              │
   │   ┌───────────────────┐  consulta PDP  ┌───────────────┐                     │
   │   │  PolicyClient     │───────────────▶│ 022-Policy    │  (default deny)     │
   │   │  (PEP, CB+cache)  │◀── allow/deny ─└───────────────┘                     │
   │   └─────────┬─────────┘                                                      │
   │             │ allow                                                          │
   │   ┌─────────▼─────────┐        ┌──────────────────────┐                      │
   │   │  SchemaRegistry   │◀──────▶│ DdlConventionValidator│                     │
   │   │ (donos, PII, ver) │        └──────────────────────┘                      │
   │   └─────┬───────┬─────┘                                                      │
   │         │       │                                                            │
   │  ┌──────▼────┐  │  ┌──────────────────┐   ┌─────────────────┐                │
   │  │ Migration │  │  │ RlsPolicyManager │   │ VectorIndex     │                │
   │  │ Engine    │  │  │ (RLS + roles)    │   │ Manager (pgvec) │                │
   │  │ (FSM §4)  │  │  └──────────────────┘   └─────────────────┘                │
   │  └──────┬────┘  │  ┌──────────────────┐   ┌─────────────────┐                │
   │         │       └─▶│ PartitionManager │──▶│ GraphStore      │                │
   │         │          └────────┬─────────┘   │ Manager (AGE)   │                │
   │         │                   │             └─────────────────┘                │
   │         │          ┌────────▼─────────┐   ┌─────────────────┐                │
   │         │          │ RetentionEnforcer│──▶│ ErasureCoord.   │──▶ 025-Audit    │
   │         │          └────────┬─────────┘   └─────────────────┘                │
   │         │                   │                                                │
   │  ┌──────▼───────────┐  ┌────▼──────────┐  ┌──────────────────┐               │
   │  │ ConnectionPool   │  │ Backup        │  │ Replication      │──▶ 027-Cluster │
   │  │ Gateway (PgB.)   │  │ Orchestrator  │  │ Supervisor       │               │
   │  └──────┬───────────┘  └────┬──────────┘  └────────┬─────────┘               │
   │         │  ┌───────────────┐│                      │                          │
   │         └─▶│ QueryGovernor ││                      │                          │
   │            └───────┬───────┘│                      │                          │
   │  ┌─────────────────▼────────▼──────────────────────▼──────────┐               │
   │  │  EventEmitter (Outbox platform.outbox → JetStream)          │              │
   │  └─────────────────────────────┬───────────────────────────────┘              │
   │  ┌──────────────────────────────────────────────────────────┐                 │
   │  │ DbTelemetry (OTel spans/metrics/logs + auditoria 025)     │                 │
   │  └──────────────────────────────────────────────────────────┘                 │
   └────────┬──────────────────┬───────────────────┬───────────────┬──────────────┘
            ▼                  ▼                   ▼               ▼
   PostgreSQL 16 (primário)  Réplicas (RO)   MinIO (WAL/backup)  NATS aios._platform.database.*
   + pgvector + AGE
```

---

## 3. Modelo de Dados / Entidades Canônicas

> Alinhado à RFC-0001 §5.1 (URN `urn:aios:<tenant>:<tipo>:<id>`, `<id>` = ULID). Toda
> tabela multi-tenant DEVE ter `tenant_id` e Row-Level Security. Tipos são do
> PostgreSQL 16. As entidades abaixo são as **do próprio módulo** (schema `platform`);
> os schemas de domínio pertencem aos módulos donos (§1.3 NR-01) e são apenas
> **catalogados** aqui.
>
> As entidades de plataforma (registro de schema, migração, política de partição/retenção,
> catálogo de backup) são **metadados globais**, não dados de um tenant final: elas usam
> o tenant reservado **`_platform`** (convenção estabelecida em `../004-API/_DESIGN_BRIEF.md` §6,
> a ser ratificada em ADR-0050). `platform.erasure_request` é a exceção: refere-se a um
> tenant real e é submetida a RLS.

### 3.1 Entidade `SchemaObject` — tabela `platform.schema_registry`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:_platform:schemaobject:<ULID>`. |
| `schema_name` | `text` | NOT NULL | Schema PostgreSQL (ex.: `kernel`, `memory`). |
| `table_name` | `text` | NOT NULL, UNIQUE(schema_name,table_name) | Tabela catalogada. |
| `owner_module` | `text` | NOT NULL | Módulo dono (`006-Kernel`, `010-Memory`, …). |
| `multi_tenant` | `boolean` | NOT NULL | Se verdadeiro, `tenant_id` + RLS são OBRIGATÓRIOS. |
| `rls_enabled` | `boolean` | NOT NULL default false | Espelho verificado do estado real no catálogo do PG. |
| `data_class` | `text` | NOT NULL, CHECK ∈ {public, internal, confidential, pii} | Classificação para LGPD/GDPR (§12.3). |
| `partition_strategy` | `text` | NOT NULL, CHECK ∈ {none, range_time, hash_tenant} | Estratégia física (§3.4). |
| `retention_ref` | `uuid` | NULL, FK→platform.retention_policy.id | Política de retenção aplicável. |
| `schema_version` | `text` | NOT NULL | Última migração aplicada a esta tabela. |
| `row_estimate` | `bigint` | NOT NULL default 0 | Cardinalidade estimada (coletada por job). |
| `bytes_estimate` | `bigint` | NOT NULL default 0 | Tamanho em disco (base para custo, `026`). |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Criação (RFC 3339 UTC). |
| `updated_at` | `timestamptz` | NOT NULL | Última mutação. |

Índices: `PK(urn)`; `UNIQUE(schema_name, table_name)`; `btree(owner_module)`;
`btree(data_class)` (varredura de conformidade).

### 3.2 Entidade `Migration` — tabela `platform.migration`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:_platform:migration:<ULID>`. |
| `version` | `text` | NOT NULL, UNIQUE | Versão ordenável `YYYYMMDDHHMMSS_<slug>`. |
| `owner_module` | `text` | NOT NULL | Módulo que submeteu a migração. |
| `state` | `text` | NOT NULL, CHECK ∈ MigrationState | Estado da FSM (§4). |
| `phase` | `text` | NOT NULL, CHECK ∈ {expand, migrate, contract} | Fase do padrão expand/contract. |
| `up_sql_digest` | `text` | NOT NULL | SHA-256 do script *up* (imutabilidade do conteúdo aplicado). |
| `down_sql_digest` | `text` | NULL | SHA-256 do script *down*, quando reversível. |
| `reversible` | `boolean` | NOT NULL default false | `contract` destrutivo NÃO é reversível. |
| `blocking_estimate_ms` | `int` | NOT NULL default 0 | Tempo estimado de bloqueio (guarda T-03). |
| `dry_run_at` | `timestamptz` | NULL | Momento do *dry-run* bem-sucedido em réplica. |
| `applied_at` | `timestamptz` | NULL | Momento da aplicação. |
| `duration_ms` | `int` | NULL | Duração real da aplicação. |
| `applied_by` | `text` | NOT NULL | Identidade (URN de serviço/operador) que aplicou. |
| `failure_code` | `text` | NULL | `AIOS-MIGRATION-*` quando `Failed`. |
| `superseded_by` | `text` | NULL, FK→platform.migration.urn | Migração corretiva (forward-fix). |
| `occ_version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Submissão. |
| `updated_at` | `timestamptz` | NOT NULL | Última transição. |

Índices: `PK(urn)`; `UNIQUE(version)`; `btree(state)`; `btree(owner_module, applied_at)`.

> **Imutabilidade:** linhas em estado terminal (`Applied`, `RolledBack`, `Superseded`)
> NÃO DEVEM ser atualizadas exceto para preencher `superseded_by`. Correções são
> **novas migrações** (forward-only), nunca reescrita do histórico.

### 3.3 Entidade `RetentionPolicy` — tabela `platform.retention_policy`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `uuid` | PK | Identidade da política. |
| `name` | `text` | NOT NULL, UNIQUE | Nome legível (`audit-7y`, `syscall-log-90d`). |
| `ttl` | `interval` | NOT NULL | Tempo de vida do registro. |
| `basis_column` | `text` | NOT NULL default `created_at` | Coluna temporal base do TTL. |
| `method` | `text` | NOT NULL, CHECK ∈ {drop_partition, delete_batch, tokenize} | Mecanismo de expurgo. |
| `legal_hold` | `boolean` | NOT NULL default false | Se verdadeiro, o expurgo é suspenso. |
| `legal_basis` | `text` | NULL | Base legal (LGPD art. 7º/GDPR art. 6º) quando `data_class='pii'`. |
| `batch_size` | `int` | NOT NULL default 10000 | Lote de `delete_batch` (evita bloqueio longo). |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |
| `updated_at` | `timestamptz` | NOT NULL | Última mutação. |

Índices: `PK(id)`; `UNIQUE(name)`; `btree(legal_hold)`.

### 3.4 Entidade `PartitionPolicy` — tabela `platform.partition_policy`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `uuid` | PK | Identidade da política. |
| `schema_object_urn` | `text` | NOT NULL, FK→platform.schema_registry.urn | Tabela particionada. |
| `strategy` | `text` | NOT NULL, CHECK ∈ {range_time, hash_tenant} | Estratégia. |
| `interval` | `interval` | NULL | Largura da partição temporal (`1 day`, `1 month`). |
| `hash_modulus` | `int` | NULL, CHECK > 0 | Nº de partições hash (potência de 2 recomendada). |
| `precreate_ahead` | `int` | NOT NULL default 7 | Quantas partições futuras manter criadas. |
| `detach_before_drop` | `boolean` | NOT NULL default true | *Detach* antes de `DROP` (janela de resgate). |
| `version` | `bigint` | NOT NULL default 0 | OCC. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |

Índices: `PK(id)`; `UNIQUE(schema_object_urn)`; `btree(strategy)`.

### 3.5 Entidade `ErasureRequest` — tabela `platform.erasure_request`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:<tenant>:erasurerequest:<ULID>`. |
| `tenant_id` | `text` | NOT NULL, RLS | Tenant titular do dado. |
| `subject_urn` | `text` | NOT NULL | URN do titular (agente, usuário, item). |
| `scope` | `text` | NOT NULL, CHECK ∈ {subject, tenant} | Alcance do expurgo. |
| `status` | `text` | NOT NULL, CHECK ∈ {received, authorized, executing, completed, rejected} | Situação. |
| `tables_affected` | `jsonb` | NOT NULL default '[]' | Tabelas resolvidas pelo `ErasureCoordinator`. |
| `rows_erased` | `bigint` | NOT NULL default 0 | Comprovante quantitativo. |
| `receipt_hash` | `text` | NULL | Hash do comprovante enviado a `025-Audit`. |
| `idempotency_key` | `text` | NULL, UNIQUE(tenant_id,idempotency_key) | RFC-0001 §5.5. |
| `requested_at` | `timestamptz` | NOT NULL | Recebimento. |
| `completed_at` | `timestamptz` | NULL | Conclusão. |

Índices: `PK(urn)`; `btree(tenant_id, status)`; `UNIQUE(tenant_id, idempotency_key)`.

### 3.6 Entidade `BackupRecord` — tabela `platform.backup_catalog`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `urn` | `text` | PK | `urn:aios:_platform:backup:<ULID>`. |
| `kind` | `text` | NOT NULL, CHECK ∈ {full, incremental, wal_segment} | Tipo do artefato. |
| `object_uri` | `text` | NOT NULL | Localização em MinIO (`s3://aios-backups/…`). |
| `lsn_start` | `pg_lsn` | NOT NULL | LSN inicial coberto. |
| `lsn_end` | `pg_lsn` | NULL | LSN final coberto. |
| `size_bytes` | `bigint` | NOT NULL | Tamanho do artefato. |
| `checksum` | `text` | NOT NULL | SHA-256 do artefato. |
| `verified_at` | `timestamptz` | NULL | Última verificação por *restore drill*. |
| `expires_at` | `timestamptz` | NOT NULL | Fim da retenção do backup. |
| `created_at` | `timestamptz` | NOT NULL | Criação. |

Índices: `PK(urn)`; `btree(kind, created_at DESC)`; `btree(expires_at)`.

### 3.7 Entidade `VectorIndexSpec` — tabela `platform.vector_index`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `uuid` | PK | Identidade da especificação. |
| `schema_object_urn` | `text` | NOT NULL, FK→platform.schema_registry.urn | Tabela com coluna vetorial. |
| `column_name` | `text` | NOT NULL, UNIQUE(schema_object_urn,column_name) | Coluna `vector(N)`. |
| `dimensions` | `int` | NOT NULL, CHECK BETWEEN 1 AND 16000 | Dimensionalidade declarada (fixa por coluna). |
| `index_type` | `text` | NOT NULL, CHECK ∈ {hnsw, ivfflat, none} | Tipo de índice. |
| `distance_op` | `text` | NOT NULL, CHECK ∈ {cosine, l2, inner_product} | Operador de distância. |
| `params` | `jsonb` | NOT NULL default '{}' | `{m, ef_construction}` (HNSW) ou `{lists}` (IVFFlat). |
| `target_recall` | `numeric(4,3)` | NOT NULL default 0.950 | Recall@10 alvo (NFR-006). |
| `last_reindex_at` | `timestamptz` | NULL | Última reindexação online. |
| `version` | `bigint` | NOT NULL default 0 | OCC. |

Índices: `PK(id)`; `UNIQUE(schema_object_urn, column_name)`.

### 3.8 Entidade `OutboxMessage` — contrato canônico `<schema>.outbox`

> Este é o **contrato de tabela** que todo módulo que usa o padrão Outbox DEVE
> instanciar em seu próprio schema (ex.: `kernel.outbox`, cf. `../006-Kernel/_DESIGN_BRIEF.md` §3.4).
> O 005 define a forma física; o conteúdo do envelope é a RFC-0001 §5.2.

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `char(26)` | PK (ULID = `event.id`) | Id do evento CloudEvents. |
| `tenant_id` | `text` | NOT NULL, RLS | Isolamento. |
| `subject` | `text` | NOT NULL | Subject NATS (RFC-0001 §5.3). |
| `payload` | `jsonb` | NOT NULL | Envelope CloudEvents completo. |
| `published` | `boolean` | NOT NULL default false | Flag do relay. |
| `created_at` | `timestamptz` | NOT NULL | Enfileiramento. |

Índices obrigatórios: `PK(id)`; `btree(created_at) WHERE published = false` (índice
parcial — mantém a varredura do relay O(pendentes), não O(histórico)).
Particionamento: `range_time` por `created_at` com retenção curta (default 7 dias).

### 3.9 Relações (ASCII)

```
   platform.schema_registry(1) ──< platform.partition_policy(0..1)
            │       │
            │       └──> platform.retention_policy(0..1) ──[legal_hold]──╳ expurgo
            │
            ├──< platform.vector_index(*)          (colunas pgvector)
            │
            └──(catálogo, sem FK física)──> schemas de domínio:
                     kernel.* · sched.* · memory.* · context.* · tools.*
                     knowledge.* · graph.* · policy.* · audit.* · cost.*

   platform.migration(*) ──[superseded_by]──> platform.migration   (forward-fix)
   platform.erasure_request(*) ──> tabelas resolvidas em tables_affected
   platform.backup_catalog(*)  ──[lsn_start..lsn_end]──> janela de PITR contínua
   <schema>.outbox(*)          ──> NATS/JetStream (relay)
```

---

## 4. Máquina de Estados Canônica da `Migration`

> A entidade com ciclo de vida governado neste módulo é a **migração de schema**.
> As demais entidades (`schema_registry`, `retention_policy`, `partition_policy`,
> `vector_index`) são configurações versionadas por OCC, sem FSM própria;
> `erasure_request` possui um ciclo simples declarado em §3.5.

### 4.1 Estados (`MigrationState`)

| Estado | Descrição | Terminal? |
|--------|-----------|-----------|
| `Draft` | Migração submetida pelo módulo dono; ainda não validada. | não |
| `Validated` | Aprovada pelo `DdlConventionValidator` (lint + revisão). | não |
| `DryRun` | Aplicada com sucesso em réplica/ambiente de ensaio; duração medida. | não |
| `Applying` | Em aplicação no primário, sob *advisory lock* global. | não |
| `Applied` | Aplicada com sucesso; `schema_registry` atualizado. | **sim** |
| `Failed` | Falhou na validação, no *dry-run* ou na aplicação; transação revertida. | **sim** |
| `RolledBack` | Migração reversível revertida deliberadamente pelo script *down*. | **sim** |
| `Superseded` | Substituída por uma migração corretiva posterior (forward-fix). | **sim** |

### 4.2 Transições, gatilhos e guardas

| # | De → Para | Gatilho | Guarda (DEVE ser verdadeira) |
|---|-----------|---------|------------------------------|
| T-01 | ∅ → `Draft` | `SubmitMigration` | `version` única ∧ `owner_module` registrado ∧ digest do script calculado. |
| T-02 | `Draft` → `Validated` | `ValidateMigration` | Lint sem violação bloqueante ∧ toda tabela multi-tenant criada tem `tenant_id`+RLS+retenção declarada. |
| T-03 | `Draft` → `Failed` | validação reprovada | Violação de convenção (`AIOS-MIGRATION-0002`) ou `blocking_estimate_ms` > `db.migration.max_block_ms`. |
| T-04 | `Validated` → `DryRun` | `DryRunMigration` | Ensaio concluído em réplica sem erro ∧ `duration_ms` medido. |
| T-05 | `Validated`/`DryRun` → `Failed` | erro no ensaio | Script falhou ou excedeu `db.migration.timeout_ms`. |
| T-06 | `DryRun` → `Applying` | `ApplyMigration` | Capability `db:migration:apply` concedida pelo PDP ∧ *advisory lock* global obtido ∧ nenhuma outra migração em `Applying` ∧ (janela permitida ∨ migração não bloqueante). |
| T-07 | `Applying` → `Applied` | commit bem-sucedido | DDL commitado ∧ `schema_registry.schema_version` atualizado ∧ evento `migration.applied` no Outbox. |
| T-08 | `Applying` → `Failed` | erro/timeout na aplicação | Transação revertida integralmente ∧ *advisory lock* liberado ∧ `failure_code` preenchido. |
| T-09 | `Applied` → `RolledBack` | `RollbackMigration` | `reversible = true` ∧ capability `db:migration:rollback` ∧ nenhuma migração posterior dependente aplicada. |
| T-10 | `Applied`/`Failed` → `Superseded` | nova migração corretiva aplicada | Existe migração posterior em `Applied` que referencia esta em `superseded_by`. |

**Invariantes:**
(I1) No máximo **uma** migração em `Applying` por cluster, garantida por
`pg_advisory_lock` global — nunca por convenção de processo.
(I2) `Applying` é **atômica**: ou a transação DDL commita inteira, ou o estado
retorna a `Failed` sem efeito residual (PostgreSQL possui DDL transacional).
(I3) Migração em fase `contract` com `reversible = false` NÃO PODE transitar para
`RolledBack`; a correção é forward-only (T-10).
(I4) Toda transição para estado terminal emite exatamente um evento
`aios._platform.database.migration.<acao>` via Outbox.
(I5) `up_sql_digest` de uma migração `Applied` é imutável; alterar o script exige
nova `version`.
(I6) Nenhuma migração alcança `Applied` sem ter passado por `DryRun` quando
`db.migration.require_dry_run = true` (default).

### 4.3 Diagrama de estados (ASCII)

```
        SubmitMigration (T-01)
   ∅ ──────────────────────────▶ ┌─────────┐
                                  │  Draft  │
                                  └────┬────┘
                validate (T-02)        │        lint/limite reprovado (T-03)
             ┌────────────────────────┤─────────────────────────────┐
             ▼                         │                             ▼
       ┌───────────┐                   │                       ┌──────────┐
       │ Validated │                   │                       │  Failed  │◀── erro no ensaio (T-05)
       └─────┬─────┘                   │                       └────┬─────┘
             │ dry-run (T-04)          │                            │ forward-fix
             ▼                         │                            │  (T-10)
       ┌───────────┐                   │                            │
       │  DryRun   │                   │                            │
       └─────┬─────┘                   │                            │
             │ apply (T-06, advisory lock global)                   │
             ▼                                                       │
       ┌───────────┐   erro/timeout (T-08)                           │
       │ Applying  │──────────────────────────▶ Failed               │
       └─────┬─────┘                                                 │
             │ commit (T-07)                                         │
             ▼                                                       ▼
       ┌───────────┐   rollback (T-09, só se reversible)   ┌──────────────┐
       │  Applied  │──────────────────────────▶┌──────────┐│  Superseded  │ (terminal)
       └─────┬─────┘                            │RolledBack│└──────────────┘
             │  forward-fix (T-10)              │(terminal)│        ▲
             └──────────────────────────────────┴──────────┴────────┘
```

---

## 5. Superfície de API (REST e gRPC)

> Autenticação/erros/idempotência/versionamento seguem RFC-0001 §5.4–§5.7. Pacote
> gRPC: `aios.database.v1`. Base REST: `/v1/database`. Toda mutação DEVE aceitar
> `Idempotency-Key` e os cabeçalhos de correlação (RFC-0001 §5.6). **Esta é uma API
> administrativa**: não há acesso de dados de domínio por ela — módulos consumidores
> falam com o PostgreSQL diretamente através do `ConnectionPoolGateway`, sob as
> convenções e credenciais que este módulo governa.

### 5.1 Operações

| Operação | REST | gRPC (rpc) | Idempotente | Notas |
|----------|------|-----------|-------------|-------|
| Submeter migração | `POST /v1/database/migrations` | `SubmitMigration` | sim (key) | Cria em `Draft` (T-01). |
| Validar migração | `POST /v1/database/migrations/{urn}:validate` | `ValidateMigration` | sim | T-02/T-03. |
| Ensaiar migração | `POST /v1/database/migrations/{urn}:dry-run` | `DryRunMigration` | sim | T-04, executa em réplica. |
| Aplicar migração | `POST /v1/database/migrations/{urn}:apply` | `ApplyMigration` | sim (key) | T-06/T-07; PEP obrigatório. |
| Reverter migração | `POST /v1/database/migrations/{urn}:rollback` | `RollbackMigration` | sim | T-09; só se `reversible`. |
| Listar/ler migrações | `GET /v1/database/migrations[/{urn}]` | `ListMigrations` / `GetMigration` | sim (leitura) | Histórico imutável. |
| Ler registro de schema | `GET /v1/database/schemas[/{schema}/{table}]` | `GetSchemaObject` | sim (leitura) | Catálogo + classificação. |
| Auditar conformidade | `POST /v1/database/schemas:audit` | `AuditSchemas` | sim (leitura) | Detecta RLS desabilitada, retenção ausente. |
| Definir retenção | `PUT /v1/database/retention-policies/{name}` | `PutRetentionPolicy` | sim | Requer `legal_basis` se `data_class='pii'`. |
| Garantir partições | `POST /v1/database/partitions:ensure` | `EnsurePartitions` | sim | Pré-criação idempotente. |
| Listar partições | `GET /v1/database/partitions` | `ListPartitions` | sim (leitura) | Inclui expiradas pendentes de *drop*. |
| Solicitar expurgo | `POST /v1/database/erasure-requests` | `RequestErasure` | sim (key) | Direito ao esquecimento (§12.3). |
| Consultar expurgo | `GET /v1/database/erasure-requests/{urn}` | `GetErasureRequest` | sim (leitura) | Comprovante e contagem. |
| Listar backups | `GET /v1/database/backups` | `ListBackups` | sim (leitura) | Catálogo + janela de PITR. |
| Verificar backup | `POST /v1/database/backups/{urn}:verify` | `VerifyBackup` | sim | *Restore drill* sob demanda. |
| Restaurar (PITR) | `POST /v1/database/restore` | `RestorePointInTime` | sim (key) | Operação crítica: PEP + aprovação dupla. |
| Status de replicação | `GET /v1/database/replication` | `GetReplicationStatus` | sim (leitura) | Lag por réplica; insumo do `027`. |
| Especificar índice vetorial | `PUT /v1/database/vector-indexes/{schema}/{table}/{column}` | `PutVectorIndex` | sim | Dimensão é imutável após criação. |
| Reindexar vetorial | `POST /v1/database/vector-indexes/{schema}/{table}/{column}:reindex` | `ReindexVector` | sim | Reindexação online (`CONCURRENTLY`). |

### 5.2 Catálogo de códigos de erro

> Formato RFC-0001 §5.4: `AIOS-<DOMINIO>-<NNNN>`. Domínios reservados por este módulo:
> **`DB`** (0001–0099) e **`MIGRATION`** (0001–0099). Registro no catálogo mantido por
> `../004-API/` (RFC-0001 §8).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-DB-0001` | 404 | não | Objeto de schema/partição/backup não encontrado. |
| `AIOS-DB-0002` | 403 | não | Operação administrativa negada pelo PDP (*default deny*). |
| `AIOS-DB-0003` | 403 | não | Tenant divergente do contexto autenticado (violação de RLS). |
| `AIOS-DB-0004` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-DB-0005` | 503 | sim | Primário indisponível; escrita rejeitada (leitura PODE degradar para réplica). |
| `AIOS-DB-0006` | 503 | sim | Pool de conexões esgotado para o serviço chamador (bulkhead). |
| `AIOS-DB-0007` | 504 | sim | `statement_timeout` excedido; consulta abortada pelo `QueryGovernor`. |
| `AIOS-DB-0008` | 409 | não | Violação de convenção física ao registrar tabela (sem `tenant_id`/RLS/retenção). |
| `AIOS-DB-0009` | 422 | não | Política de retenção sem `legal_basis` para tabela classificada como `pii`. |
| `AIOS-DB-0010` | 423 | não | Expurgo bloqueado por `legal_hold` ativo. |
| `AIOS-DB-0011` | 409 | não | Dimensão de coluna vetorial imutável: alteração exige nova coluna + migração. |
| `AIOS-DB-0012` | 507 | não | Espaço em disco/limite de armazenamento do tenant excedido (limite de `026`). |
| `AIOS-DB-0013` | 503 | sim | Lag de replicação acima do limiar; leitura obsoleta recusada. |
| `AIOS-MIGRATION-0001` | 409 | sim | Já existe migração em `Applying` (advisory lock ocupado). |
| `AIOS-MIGRATION-0002` | 422 | não | Migração reprovada no lint de convenções (T-03). |
| `AIOS-MIGRATION-0003` | 409 | não | Transição de estado inválida (viola a FSM §4). |
| `AIOS-MIGRATION-0004` | 409 | não | `version` de migração duplicada. |
| `AIOS-MIGRATION-0005` | 504 | sim | Timeout na aplicação; transação revertida (T-08). |
| `AIOS-MIGRATION-0006` | 409 | não | Rollback recusado: migração não reversível ou há migração posterior dependente. |
| `AIOS-MIGRATION-0007` | 412 | não | *Dry-run* obrigatório ausente (`db.migration.require_dry_run`). |
| `AIOS-MIGRATION-0008` | 409 | não | Digest do script divergente do registrado (I5). |

---

## 6. Catálogo de Eventos NATS

> Envelope CloudEvents e convenção de subjects: RFC-0001 §5.2–§5.3
> (`aios.<tenant>.<dominio>.<entidade>.<acao>`). `<dominio>` = **`database`**, novo
> valor a ser registrado no registro de domínios mantido por `../004-API/` conforme
> RFC-0001 §8 (ADR-0050). Entrega at-least-once via JetStream; consumidores DEVEM
> deduplicar por `event.id`.
>
> Eventos de **plataforma** (migração, partição, backup, replicação) usam o tenant
> reservado **`_platform`**; eventos que refletem dado de um tenant (expurgo, limite de
> armazenamento) usam o `<tenant>` real.

### 6.1 Eventos emitidos (produtor: 005-Database)

| Subject | `type` | Quando | Stream |
|---------|--------|--------|--------|
| `aios._platform.database.migration.applied` | `aios.database.migration.applied` | Migração → `Applied` (T-07). | DB_PLATFORM (durável) |
| `aios._platform.database.migration.failed` | `aios.database.migration.failed` | Migração → `Failed` (T-03/T-05/T-08). | DB_PLATFORM |
| `aios._platform.database.migration.rolledback` | `aios.database.migration.rolledback` | Migração → `RolledBack` (T-09). | DB_PLATFORM |
| `aios._platform.database.schema.registered` | `aios.database.schema.registered` | Nova tabela catalogada no `schema_registry`. | DB_PLATFORM |
| `aios._platform.database.partition.created` | `aios.database.partition.created` | Partição futura pré-criada. | DB_PLATFORM |
| `aios._platform.database.partition.dropped` | `aios.database.partition.dropped` | Partição expirada removida. | DB_PLATFORM |
| `aios._platform.database.backup.completed` | `aios.database.backup.completed` | Backup full/incremental concluído e catalogado. | DB_PLATFORM |
| `aios._platform.database.backup.verified` | `aios.database.backup.verified` | *Restore drill* concluído com sucesso. | DB_PLATFORM |
| `aios._platform.database.replication.lagging` | `aios.database.replication.lagging` | Lag > `db.replication.max_lag_ms`. | DB_HEALTH |
| `aios._platform.database.replication.promoted` | `aios.database.replication.promoted` | Réplica promovida a primário (executado com `027`). | DB_HEALTH |
| `aios._platform.database.compliance.violated` | `aios.database.compliance.violated` | Auditoria achou tabela multi-tenant sem RLS ou sem retenção. | DB_HEALTH |
| `aios.<tenant>.database.retention.purged` | `aios.database.retention.purged` | Expurgo por TTL concluído (volume e tabelas). | DB_RETENTION |
| `aios.<tenant>.database.erasure.completed` | `aios.database.erasure.completed` | Direito ao esquecimento concluído (com `receipt_hash`). | DB_RETENTION |
| `aios.<tenant>.database.storage.threshold` | `aios.database.storage.threshold` | Uso de armazenamento do tenant cruzou limiar (ex.: 80%). | DB_HEALTH |

### 6.2 Eventos consumidos

| Subject assinado | Produtor | Ação do 005-Database |
|-------------------|----------|----------------------|
| `aios._platform.cluster.failover.requested` | `027-Cluster` | Coordena promoção de réplica e reconfiguração do pool. |
| `aios._platform.deployment.rollout.started` | `028-Deployment` | Congela migrações `contract` durante a janela de rollout. |
| `aios.<tenant>.policy.bundle.updated` | `022-Policy` | Invalida cache de decisões do `PolicyClient`. |
| `aios.<tenant>.cost.budget.updated` | `026-Cost-Optimizer` | Atualiza limite de armazenamento por tenant (`AIOS-DB-0012`). |
| `aios.<tenant>.audit.legalhold.applied` | `025-Audit` | Ativa `legal_hold` e suspende expurgo das tabelas afetadas. |
| `aios.<tenant>.memory.item.forgotten` | `010-Memory` | Enfileira expurgo físico do item e de seus vetores. |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito | Prioridade | Critério de aceite |
|----|-----------|-----------|--------------------|
| FR-001 | Manter o catálogo autoritativo de schemas, donos e classificação de dados. | Must | `GET /v1/database/schemas` lista 100% das tabelas de domínio com `owner_module` e `data_class`. |
| FR-002 | Rejeitar criação de tabela multi-tenant sem `tenant_id`, RLS e retenção declarada. | Must | Migração violadora → `Failed` com `AIOS-MIGRATION-0002`; teste de lint cobre os 3 casos. |
| FR-003 | Aplicar migrações forward-only sob *advisory lock* global, uma por vez. | Must | Duas aplicações concorrentes: a segunda recebe `AIOS-MIGRATION-0001`. |
| FR-004 | Exigir *dry-run* em réplica antes da aplicação, quando configurado. | Must | `apply` sem `dry_run_at` → `AIOS-MIGRATION-0007`. |
| FR-005 | Reverter apenas migrações marcadas como reversíveis e sem dependentes. | Should | Rollback de `contract` destrutivo → `AIOS-MIGRATION-0006`. |
| FR-006 | Garantir isolamento multi-tenant por RLS em toda tabela multi-tenant. | Must | Teste: sessão com `tenant_id` A não lê nenhuma linha de B, em 100% das tabelas. |
| FR-007 | Pré-criar partições futuras e remover as expiradas conforme política. | Must | Sempre ≥ `precreate_ahead` partições futuras existentes; nenhuma partição além do TTL. |
| FR-008 | Executar retenção preferindo *drop de partição* a `DELETE`. | Must | Tabelas `range_time` expurgam sem *bloat*; `pg_stat` sem crescimento de tuplas mortas. |
| FR-009 | Executar o direito ao esquecimento com comprovante rastreável. | Must | `erasure.completed` emitido com `rows_erased` e `receipt_hash` registrado em `025`. |
| FR-010 | Suspender expurgo sob `legal_hold`. | Must | Pedido durante *hold* → `AIOS-DB-0010`; nenhuma linha removida. |
| FR-011 | Manter WAL archiving contínuo e catálogo de backups com janela de PITR. | Must | `GET /v1/database/backups` informa janela contínua sem lacuna de LSN. |
| FR-012 | Verificar backups por *restore drill* automatizado periódico. | Must | `backup.verified` emitido no período configurado; falha gera alerta. |
| FR-013 | Governar índices vetoriais (tipo, parâmetros, operador) e permitir reindexação online. | Must | `PutVectorIndex` cria índice; `reindex` executa `CONCURRENTLY` sem bloquear escrita. |
| FR-014 | Governar grafos AGE com isolamento por tenant equivalente ao RLS. | Should | Travessia em grafo de A não retorna vértices de B. |
| FR-015 | Aplicar cota de conexões e `statement_timeout` por serviço. | Must | Excesso → `AIOS-DB-0006`; consulta longa → `AIOS-DB-0007`. |
| FR-016 | Recusar leitura em réplica com lag acima do limiar. | Should | Lag > limiar → `AIOS-DB-0013` ou redirecionamento ao primário. |
| FR-017 | Consultar o PDP antes de toda operação administrativa. | Must | Operação sem capability → `AIOS-DB-0002`; auditoria registra `deny`. |
| FR-018 | Emitir todos os eventos da §6 via Outbox transacional. | Must | Sem evento perdido em teste de crash entre commit e publish. |
| FR-019 | Publicar o contrato canônico da tabela `outbox` e validar suas instâncias. | Should | Auditoria detecta `outbox` sem índice parcial ou sem retenção. |
| FR-020 | Reportar consumo de armazenamento por tenant como insumo de custo. | Should | `bytes_estimate` atualizado ≤ 24h; evento `storage.threshold` no limiar. |

### 7.2 Requisitos Não-Funcionais (NFR) com SLO/SLI

| ID | Atributo | Meta (SLO) | SLI / método de verificação |
|----|----------|-----------|-----------------------------|
| NFR-001 | Desempenho (leitura por chave) | p99 ≤ **5 ms** para busca por PK/URN em tabela quente. | Histograma `aios_db_query_latency_ms{kind="pk"}`. |
| NFR-002 | Desempenho (escrita transacional) | p99 ≤ **15 ms** para `INSERT`/`UPDATE` de linha única com commit. | `aios_db_query_latency_ms{kind="write"}`. |
| NFR-003 | Desempenho (busca vetorial) | p99 ≤ **50 ms** para top-K (K=10) em índice HNSW com ≥ 10⁷ vetores. | `aios_db_vector_search_latency_ms`; benchmark em `Benchmark.md`. |
| NFR-004 | Throughput | ≥ **20.000 transações/s** no primário (mix 80/20 leitura/escrita) com pooling. | Teste de carga (pgbench + carga sintética AIOS). |
| NFR-005 | Disponibilidade | ≥ **99,95%** do serviço de dados (primário + failover automático). | Uptime probe; error budget mensal. |
| NFR-006 | Qualidade de busca vetorial | Recall@10 ≥ **0,95** contra busca exata, por índice. | Job de amostragem comparando HNSW vs. *exact scan*. |
| NFR-007 | Escalabilidade | Suportar ≥ **10⁶** agentes e ≥ **10⁹** linhas em tabelas particionadas com crescimento sub-linear de latência. | Teste de escala `Scalability.md`. |
| NFR-008 | Durabilidade | Perda de transação commitada = **0** (`synchronous_commit=on` + réplica síncrona no quórum). | Chaos test: kill do primário sob carga. |
| NFR-009 | Recuperação | **RTO ≤ 15 min**, **RPO ≤ 5 min** (alvo operacional de RPO: ≤ 1 min via WAL streaming). | DR drill trimestral; ver §9. |
| NFR-010 | Migração sem downtime | Migração `expand` NÃO DEVE bloquear escrita por mais de **200 ms**; `contract` executado apenas fora da janela de rollout. | `aios_db_migration_block_ms`; ADR-0053. |
| NFR-011 | Isolamento multi-tenant | **0** vazamento entre tenants; 100% das tabelas multi-tenant com RLS ativa. | Auditoria contínua (`schemas:audit`) + teste de penetração de RLS. |
| NFR-012 | Consistência | Isolamento `READ COMMITTED` como default; `REPEATABLE READ` disponível; OCC por `version` sem locks pessimistas no caminho quente. | Teste de concorrência; contagem de deadlocks ≈ 0. |
| NFR-013 | Lag de replicação | p95 ≤ **500 ms**; leitura obsoleta recusada acima de `db.replication.max_lag_ms`. | `aios_db_replication_lag_ms`. |
| NFR-014 | Observabilidade | 100% das operações administrativas com trace OTel e correlação (`trace_id`, `tenant_id`). | Inspeção de spans; cobertura de campos. |
| NFR-015 | Idempotência | Repetições de mutação administrativa produzem efeito único em 100% dos casos. | Teste de replay com `Idempotency-Key`. |
| NFR-016 | Eficiência de armazenamento | *Bloat* de tabela ≤ **20%**; autovacuum ajustado por tabela de alta rotatividade. | `aios_db_table_bloat_ratio`. |

---

## 8. Chaves de Configuração Principais

> Escopo: `global` (cluster), `tenant`, `table`. Recarregável indica *hot reload*.
> Prefixo de env var: `AIOS_DB_`.

| Chave | Default | Faixa | Escopo | Recarregável | Descrição |
|-------|---------|-------|--------|--------------|-----------|
| `db.pool.max_connections_per_service` | 50 | 1–500 | global | sim | Bulkhead de conexões por serviço no PgBouncer. |
| `db.pool.mode` | `transaction` | {session, transaction, statement} | global | **não** | Modo do PgBouncer; `transaction` é o default do AIOS. |
| `db.query.statement_timeout_ms` | 30000 | 100–600000 | global/tenant | sim | Timeout de consulta antes de aborto (`AIOS-DB-0007`). |
| `db.query.idle_in_tx_timeout_ms` | 60000 | 1000–600000 | global | sim | Corta transações ociosas que seguram locks. |
| `db.query.slow_threshold_ms` | 500 | 10–60000 | global | sim | Limiar de captura de plano pelo `QueryGovernor`. |
| `db.migration.require_dry_run` | `true` | {true,false} | global | sim | Exige `DryRun` antes de `Applying` (I6). |
| `db.migration.max_block_ms` | 200 | 0–5000 | global | sim | Bloqueio máximo tolerado numa migração (guarda T-03). |
| `db.migration.timeout_ms` | 900000 | 1000–3600000 | global | sim | Timeout total de aplicação (T-08). |
| `db.migration.lock_timeout_ms` | 5000 | 100–60000 | global | sim | Espera pelo *advisory lock* antes de `AIOS-MIGRATION-0001`. |
| `db.rls.enforce` | `true` | {true,false} | global | **não** | Desligar exige migração + aprovação; auditado como violação. |
| `db.partition.precreate_ahead` | 7 | 1–90 | global/table | sim | Partições futuras mantidas criadas. |
| `db.partition.drop_grace_h` | 24 | 0–720 | global/table | sim | Janela entre *detach* e `DROP` de partição. |
| `db.retention.default_ttl` | `90d` | 1d–3650d | table | sim | TTL default quando a tabela não declara política. |
| `db.retention.batch_size` | 10000 | 100–1000000 | table | sim | Lote de `delete_batch` para evitar bloqueio longo. |
| `db.retention.run_interval_min` | 60 | 5–1440 | global | sim | Frequência do `RetentionEnforcer`. |
| `db.backup.full_interval_h` | 24 | 1–168 | global | sim | Periodicidade do backup full. |
| `db.backup.wal_archive_timeout_s` | 60 | 5–600 | global | sim | Força arquivamento de WAL (limita o RPO). |
| `db.backup.retention_days` | 35 | 7–3650 | global | sim | Retenção de artefatos de backup em MinIO. |
| `db.backup.verify_interval_h` | 168 | 24–2160 | global | sim | Periodicidade do *restore drill* (FR-012). |
| `db.replication.max_lag_ms` | 1000 | 50–60000 | global | sim | Lag acima do qual a leitura em réplica é recusada. |
| `db.replication.read_from_replica` | `true` | {true,false} | global/tenant | sim | Habilita roteamento de leitura elegível. |
| `db.vector.default_index` | `hnsw` | {hnsw, ivfflat, none} | global/table | sim | Tipo default de índice vetorial. |
| `db.vector.hnsw_m` | 16 | 4–96 | table | **não** | Parâmetro `m` do HNSW (alterar exige reindexação). |
| `db.vector.hnsw_ef_construction` | 64 | 16–512 | table | **não** | Qualidade de construção do HNSW. |
| `db.vector.ef_search` | 64 | 16–1024 | global/tenant | sim | Esforço de busca (troca recall × latência). |
| `db.graph.max_traversal_depth` | 6 | 1–20 | global/tenant | sim | Profundidade máxima de travessia AGE (contenção de DoS). |
| `db.storage.tenant_quota_gb` | 100 | 1–100000 | tenant | sim | Cota de armazenamento (limite efetivo vem de `026`). |
| `db.autovacuum.scale_factor_hot` | 0.02 | 0.001–0.5 | table | sim | Autovacuum agressivo em tabelas de alta rotatividade. |

---

## 9. Modos de Falha e Recuperação

| Modo de falha | Detecção | Isolamento | Recuperação | Idempotência/Retry |
|---------------|----------|-----------|-------------|--------------------|
| Perda do primário PostgreSQL | Health check + ausência de heartbeat de replicação | Escritas rejeitadas (`AIOS-DB-0005`); leituras seguem em réplica | Promoção de réplica coordenada com `027-Cluster`; reconfiguração do PgBouncer | Escritas não commitadas são reenviadas pelo cliente com `Idempotency-Key` |
| Réplica com lag excessivo | `aios_db_replication_lag_ms` > limiar | Réplica removida do pool de leitura | Recuperação natural ou *rebuild* da réplica; evento `replication.lagging` | Leitura redirecionada ao primário |
| Migração falha no meio | Erro/timeout em `Applying` | DDL transacional reverte tudo (I2) | Estado → `Failed`; correção por nova migração (forward-fix) | *Advisory lock* liberado; reaplicação idempotente por `version` |
| Migração trava tabela além do orçamento | `aios_db_migration_block_ms` | Abortada pelo `lock_timeout` | Reescrita como `expand/contract` incremental | Nova `version`, sem efeito parcial |
| Esgotamento do pool de conexões | Fila do PgBouncer | Bulkhead por serviço: um serviço não drena o pool dos outros | `AIOS-DB-0006` com `Retry-After`; escala de réplicas de leitura | Retry com backoff exponencial |
| Consulta patológica (plano regredido) | `pg_stat_statements` + captura de plano | `statement_timeout` aborta | `QueryGovernor` sinaliza índice ausente; `ANALYZE`/reindex | Consulta é leitura: retry seguro |
| Disco cheio no primário | Alerta de espaço + falha de WAL | Escritas bloqueadas antes da corrupção | Expurgo de partições expiradas + expansão de volume; `AIOS-DB-0012` | Sem efeito parcial (transação falha inteira) |
| Corrupção de página / perda de dados | Checksums do PostgreSQL + falha de leitura | Tabela/partição isolada | **PITR** a partir do último backup + WAL até o LSN saudável | Restauração determinística por LSN |
| Falha de arquivamento de WAL | `db.backup.wal_archive_timeout_s` sem progresso | Alerta imediato (o RPO está degradado) | Reenvio para MinIO; se persistir, escrita entra em modo conservador | Segmentos de WAL são idempotentes por nome |
| Backup irrecuperável | *Restore drill* falha (FR-012) | Backup marcado como não verificado | Novo backup full imediato + investigação; alerta P1 | Verificação repetível |
| Crash entre commit e publish de evento | Relay detecta `published = false` | Estado no PG intacto | Relay reenvia; consumidores deduplicam por `event.id` | Exactly-once efetivo |
| RLS desabilitada indevidamente | Auditoria contínua (`schemas:audit`) | Evento `compliance.violated` + alerta P1 | Reabilitação imediata via migração emergencial | Operação idempotente (`ENABLE ROW LEVEL SECURITY`) |
| PDP (022) indisponível | Timeout/CB no `PolicyClient` | Bulkhead do PEP | *Fail-closed*: operação administrativa negada (`AIOS-DB-0002`) | Retry após meia-abertura do CB |
| Expurgo interrompido no meio | `erasure_request` em `executing` além do prazo | Lote parcial commitado (ordem segura de FK) | Retomada idempotente pelo `subject_urn` restante | Repetição não remove o que já não existe |

**Metas de recuperação:** **RTO ≤ 15 min**, **RPO ≤ 5 min** (alvo operacional
≤ 1 min com WAL streaming contínuo), alinhado a V-R10 da Visão e §2 da Arquitetura.
Degradação graciosa: sob indisponibilidade parcial, o módulo prioriza **preservar a
integridade e negar com segurança** (rejeitar escrita, recusar leitura obsoleta,
*fail-closed* no PEP) em vez de servir dado incorreto ou sem governança.

---

## 10. Escalabilidade e Concorrência

| Dimensão | Estratégia |
|----------|-----------|
| Particionamento temporal | Tabelas de série (`syscall_log`, `audit`, `outbox`, métricas de execução) usam **RANGE por tempo**, com pré-criação (`precreate_ahead`) e expurgo por *drop de partição* — custo de retenção O(1), sem *bloat*. |
| Particionamento por tenant | Tabelas de alta cardinalidade multi-tenant usam **HASH sobre `tenant_id`** (`hash_modulus`), distribuindo *hot tenants* e limitando o tamanho de índice por partição. |
| Sharding lógico | Alinhado ao sharding do plano de controle (`shard = hash(tenant_id, agent_id) mod N`, cf. `../006-Kernel/_DESIGN_BRIEF.md` §10): a coluna `shard_key` é chave de partição/índice, permitindo migrar shards inteiros para outro cluster sem reescrever a aplicação. |
| Leitura vs. escrita | **Réplicas de streaming** absorvem leituras elegíveis (relatórios, catálogos, buscas não críticas) sob controle de *lag* (NFR-013); escrita sempre no primário. |
| Concorrência de linha | **OCC por `version`** como padrão do AIOS; sem locks pessimistas no caminho quente. `SELECT ... FOR UPDATE` restrito a seções curtas e justificadas. |
| Conexões | PgBouncer em modo *transaction* com **bulkhead por serviço**: nenhum serviço consegue esgotar o pool alheio; o caminho quente do agente nunca espera indefinidamente por conexão. |
| Busca vetorial | HNSW por default (latência estável, sem necessidade de treino), IVFFlat para conjuntos muito grandes com tolerância a *build*; `ef_search` ajustável por tenant para trocar recall por latência. |
| Grafo (AGE) | Grafo por tenant, com profundidade máxima de travessia limitada (`db.graph.max_traversal_depth`) — evita consulta de custo explosivo como vetor de DoS. |
| Backpressure | Cota de conexões + `statement_timeout` + rejeição precoce (`AIOS-DB-0006`, `503`/`Retry-After`) em vez de degradar todos os tenants (contenção de *noisy neighbor*). |
| Manutenção sem downtime | `CREATE INDEX CONCURRENTLY`, `REINDEX CONCURRENTLY`, `ADD COLUMN` sem default volátil, backfill em lotes — codificados como regras do `DdlConventionValidator`. |
| Rumo a milhões | 10⁶ agentes ⇒ ~10⁹ linhas em tabelas de série. A combinação **partição temporal + hash por tenant + retenção por drop + índice parcial no outbox + réplicas de leitura** mantém o índice ativo pequeno (o *working set* é sempre o presente, não o histórico) e o custo por agente aproximadamente constante. Escala vertical do primário é o último recurso; o caminho de crescimento é **mais partições, mais réplicas e, no limite, mais clusters por faixa de shard** (ver `../027-Cluster/`). |

```
   Tabela lógica  memory.item                Tabela lógica  kernel.syscall_log
   ┌───────────────────────────────┐        ┌─────────────────────────────────┐
   │ HASH(tenant_id) mod 16        │        │ RANGE(created_at) por dia       │
   │  p00 p01 p02 … p15            │        │  d-0  d-1  d-2 … d-89           │
   └───────────────────────────────┘        └───────────┬─────────────────────┘
        │ índice ativo por partição                     │ drop O(1) em d-90
        ▼                                               ▼
   working set pequeno e estável            retenção sem DELETE, sem bloat
```

---

## 11. ADRs e RFCs a Propor

> **Faixa de ADR reservada a este módulo: `ADR-0050`..`ADR-0059`** (regra 005×10,
> evitando colisão entre módulos). Registrar em `../002-ADR/`. ADR-0005 (escolha do
> PostgreSQL + pgvector + AGE como plataforma de dados) é **herdada** da fundação.

| ADR | Título proposto |
|-----|-----------------|
| ADR-0050 | Schema por módulo dono, catálogo central `platform.schema_registry` e registro do domínio de eventos `database`. |
| ADR-0051 | RLS por `tenant_id` como fronteira física de isolamento multi-tenant (vs. schema por tenant vs. banco por tenant). |
| ADR-0052 | Convenções físicas obrigatórias (URN como PK, `version` para OCC, `timestamptz` UTC) e o *linter* de DDL que as impõe. |
| ADR-0053 | Migrações forward-only com padrão expand/contract e *advisory lock* global. |
| ADR-0054 | Estratégia de particionamento: RANGE temporal para séries, HASH por `tenant_id` para alta cardinalidade. |
| ADR-0055 | Retenção por *drop de partição* como mecanismo primário de expurgo (vs. `DELETE` em lote). |
| ADR-0056 | Backup contínuo (WAL archiving em MinIO) + PITR, com *restore drill* automatizado como critério de validade. |
| ADR-0057 | pgvector: HNSW como índice default, dimensão imutável por coluna e política de reindexação online. |
| ADR-0058 | Apache AGE para o grafo de conhecimento no mesmo cluster (vs. banco de grafos dedicado). |
| ADR-0059 | PgBouncer em modo *transaction* com bulkhead de conexões por serviço; domínios de erro `DB` e `MIGRATION`. |

| RFC | Título proposto | Relação |
|-----|-----------------|---------|
| RFC-0001 | Architecture Baseline & Core Contracts | **Baseline** (consumida, não redefinida). |
| RFC-0050 | AIOS Data Platform Contract (convenções físicas, RLS, contrato da tabela `outbox`, classificação de dados). | A propor por este módulo. |
| RFC-0051 | AIOS Migration Protocol (formato de migração, FSM, expand/contract, garantias de compatibilidade). | A propor por este módulo. |

---

## 12. Decisões de Segurança

### 12.1 AuthN / AuthZ

- **AuthN**: identidade de serviço estabelecida por **mTLS** interno (RFC-0001 §6);
  credenciais de banco (usuário/senha ou certificado) são custodiadas por
  `021-Security` e **rotacionadas sem downtime**. O 005 **não** emite nem valida
  tokens de usuário final; a API administrativa é exposta via `004-API`, que valida
  OAuth2/OIDC e propaga claims assinadas.
- **AuthZ**: o 005 é **PEP**; toda operação administrativa (aplicar/reverter migração,
  alterar retenção, expurgar, restaurar backup) consulta o **PDP** do `022-Policy`
  com **default deny** (`AIOS-DB-0002`). Operações de restauração exigem, além disso,
  aprovação dupla (política do `022`, verificada como *capability* distinta).
- **Privilégio mínimo no banco**: cada serviço recebe uma *role* própria com `GRANT`
  restrito ao seu schema. Nenhum serviço de runtime opera como `SUPERUSER` nem como
  dono das tabelas — apenas o `MigrationEngine` assume a *role* de DDL, e apenas
  durante uma migração autorizada.
- **Isolamento de tenant**: RLS por `tenant_id` é a fronteira física. O 005 **NÃO
  DEVE** aceitar sessão cujo `tenant` divirja do contexto autenticado
  (`AIOS-DB-0003`), e a auditoria contínua trata RLS desabilitada como violação P1.

### 12.2 Threat Model STRIDE (resumido)

| Ameaça (STRIDE) | Vetor no 005-Database | Mitigação |
|-----------------|------------------------|-----------|
| **S**poofing | Serviço se conecta com credencial de outro serviço/tenant para ler dados alheios. | mTLS + *role* por serviço + credenciais rotacionadas no cofre (`021`); `tenant` da sessão casado com o contexto autenticado (`AIOS-DB-0003`). |
| **T**ampering | Alteração direta de linhas, do histórico de migrações ou de políticas de retenção para encobrir ação. | Privilégio mínimo (sem DDL/`UPDATE` fora do schema próprio); `platform.migration` imutável em estado terminal (I5); toda operação administrativa auditada em `025`; checksums de página ativos. |
| **R**epudiation | Negar ter aplicado uma migração destrutiva ou expurgado dados. | `applied_by` + `up_sql_digest` na migração; `receipt_hash` no expurgo; auditoria imutável com `trace_id` em `025-Audit`. |
| **I**nformation disclosure | Vazamento entre tenants por query defeituosa, por backup mal protegido ou por PII em log de consulta. | **RLS** como rede de segurança independente da aplicação; backups cifrados em repouso no MinIO e cifrados em trânsito; `QueryGovernor` **NÃO DEVE** registrar valores de parâmetros de consultas em tabelas `pii` (apenas o texto normalizado). |
| **D**enial of service | Consulta de custo explosivo (travessia de grafo, *seq scan* em 10⁹ linhas), esgotamento de conexões, enchimento de disco. | `statement_timeout`, `max_traversal_depth`, bulkhead de conexões por serviço, cota de armazenamento por tenant (`AIOS-DB-0012`), rejeição precoce com `Retry-After`. |
| **E**levation of privilege | Migração maliciosa que cria função `SECURITY DEFINER`, desabilita RLS ou concede `SUPERUSER`. | `DdlConventionValidator` rejeita DDL que desabilite RLS, crie `SECURITY DEFINER` ou altere *roles*; aplicação exige capability `db:migration:apply` no PDP; auditoria contínua detecta divergência entre catálogo real e `schema_registry`. |

### 12.3 LGPD / GDPR

- **Minimização e classificação**: toda tabela é classificada (`data_class`), e
  tabelas `pii` DEVEM declarar `legal_basis` na política de retenção
  (`AIOS-DB-0009`) — sem base legal declarada, a tabela não passa no lint.
- **Retenção**: nenhum dado pessoal é retido além do TTL declarado; o expurgo padrão
  é *drop de partição*, e o registro do expurgo (volume, tabelas, momento) é
  auditável — mas **não contém o dado expurgado**.
- **Direito ao esquecimento**: `POST /v1/database/erasure-requests` resolve o grafo de
  tabelas que referenciam o titular (incluindo vetores em `pgvector` e vértices em AGE),
  executa a remoção em ordem segura de FK e emite `erasure.completed` com
  `receipt_hash`. Quando a remoção física é impossível por obrigação legal, aplica-se
  **tokenização irreversível** em vez de exclusão, e isso é registrado no comprovante.
- **Legal hold**: `legal_hold` suspende qualquer expurgo (`AIOS-DB-0010`), inclusive o
  automático por TTL — a obrigação de preservar prevalece sobre a de apagar, e o
  conflito é registrado, nunca resolvido silenciosamente.
- **Segregação**: dados de tenants distintos são isolados por RLS e por partição;
  backups herdam o mesmo escopo lógico, e a restauração parcial por tenant é suportada.
- **Cifragem**: dados em repouso cifrados no volume e nos artefatos de backup; tráfego
  cliente↔banco sempre por TLS.

---

## 13. Referências

- Visão: `../000-Vision/Vision.md`
- Arquitetura: `../001-Architecture/Architecture.md`
- Glossário: `../040-Glossary/Glossary.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Decisões: `../002-ADR/README.md`
- Template de módulo: `../_templates/MODULE_TEMPLATE.md`
- Índice do módulo: `./README.md`
- Módulos acoplados: `../004-API/`, `../006-Kernel/`, `../009-Scheduler/`,
  `../010-Memory/`, `../011-Context/`, `../018-Knowledge/`, `../019-GraphRAG/`,
  `../020-Communication/`, `../021-Security/`, `../022-Policy/`,
  `../024-Observability/`, `../025-Audit/`, `../026-Cost-Optimizer/`,
  `../027-Cluster/`, `../028-Deployment/`, `../029-Operations/`.

*Fim do Design Brief interno do módulo 005-Database. Este documento governa a geração
dos 26 documentos obrigatórios; qualquer conflito entre um documento gerado e este
brief é um defeito do documento gerado.*
