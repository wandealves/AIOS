---
Documento: ClassDiagrams
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0050, ADR-0052, ADR-0053, ADR-0057, ADR-0059
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: _DESIGN_BRIEF.md §2, Architecture.md, StateMachine.md, Database.md
---

# 005-Database — Estruturas, Interfaces e Contratos

Diagramas em ASCII das estruturas internas do `aios-database-svc` (.NET 10). Os nomes
de componente são exatamente os de `./_DESIGN_BRIEF.md` §2.1 e de `./Architecture.md` §3.

---

## 1. Visão de Pacotes

```
   Aios.Database.Api            → controllers REST + serviços gRPC (aios.database.v1)
   Aios.Database.Schema         → SchemaRegistry, DdlConventionValidator, RlsPolicyManager
   Aios.Database.Migration      → MigrationEngine (FSM), MigrationLedger
   Aios.Database.Lifecycle      → PartitionManager, RetentionEnforcer, ErasureCoordinator
   Aios.Database.Extensions     → VectorIndexManager, GraphStoreManager
   Aios.Database.Durability     → BackupOrchestrator, ReplicationSupervisor
   Aios.Database.Access         → ConnectionPoolGateway, QueryGovernor
   Aios.Database.Platform       → PolicyClient, EventEmitter, DbTelemetry
```

Regra de dependência: `Api` → (`Schema`, `Migration`, `Lifecycle`, `Extensions`,
`Durability`, `Access`) → `Platform`. **Nenhum** pacote de domínio depende de `Api`,
e `Platform` **NÃO DEVE** depender de nenhum outro (evita ciclo).

---

## 2. Diagrama de Componentes e Relações

```
  ┌───────────────────────────┐        ┌──────────────────────────────┐
  │      MigrationEngine      │───────▶│   DdlConventionValidator     │
  │───────────────────────────│        │──────────────────────────────│
  │ - ledger: MigrationLedger │        │ + Validate(sql, ctx): Report │
  │ - lock: IAdvisoryLock     │        │ - rules: IReadOnlyList<IRule>│
  │───────────────────────────│        └──────────────────────────────┘
  │ + SubmitAsync(): Migration│                     △ (implementa)
  │ + ValidateAsync(): ...    │        ┌────────────┴─────────────┐
  │ + DryRunAsync(): ...      │        │ TenantIdRequiredRule     │
  │ + ApplyAsync(): ...       │        │ RlsPolicyRequiredRule    │
  │ + RollbackAsync(): ...    │        │ RetentionDeclaredRule    │
  └────┬──────────┬───────────┘        │ UrnPrimaryKeyRule        │
       │          │                    │ TimestamptzRule          │
       │          │                    │ IndexOnForeignKeyRule    │
       │          │                    │ NoDangerousDdlRule       │
       │          │                    │ BlockingBudgetRule       │
       │          │                    └──────────────────────────┘
       │          ▼
       │   ┌───────────────────┐  usa   ┌──────────────────────┐
       │   │  SchemaRegistry   │◀──────▶│  RlsPolicyManager    │
       │   │───────────────────│        │──────────────────────│
       │   │ + Get(schema,tbl) │        │ + EnsurePolicy(tbl)  │
       │   │ + Upsert(obj)     │        │ + AuditPolicies()    │
       │   │ + Audit(): Report │        │ + EnsureRole(service)│
       │   └───┬───────────┬───┘        └──────────────────────┘
       │       │           │
       │       ▼           ▼
       │  ┌──────────────┐ ┌────────────────────┐
       │  │PartitionMgr  │ │ VectorIndexManager │
       │  │+ EnsureAhead │ │ + Put(spec)        │
       │  │+ DropExpired │ │ + ReindexAsync()   │
       │  └──────┬───────┘ │ + MeasureRecall()  │
       │         │         └────────────────────┘
       │         ▼
       │  ┌──────────────────┐   ┌────────────────────┐
       │  │ RetentionEnforcer│──▶│ ErasureCoordinator │
       │  │ + RunCycle()     │   │ + Execute(request) │
       │  └──────────────────┘   └────────────────────┘
       │
       ▼
  ┌───────────────┐   ┌──────────────┐   ┌──────────────┐
  │ PolicyClient  │   │ EventEmitter │   │ DbTelemetry  │
  │ + Decide(req) │   │ + Enqueue(e) │   │ + Span(name) │
  └───────────────┘   └──────────────┘   └──────────────┘

  ┌────────────────────────┐    ┌────────────────────┐    ┌───────────────────────┐
  │ ConnectionPoolGateway  │───▶│ ReplicationSuperv. │    │ BackupOrchestrator    │
  │ + Acquire(service,rw)  │    │ + LagMs(replica)   │    │ + RunFull()           │
  │ + Route(query): Target │    │ + EligibleReaders()│    │ + ArchiveWal()        │
  └───────────┬────────────┘    │ + Promote(replica) │    │ + VerifyRestore(urn)  │
              ▼                 └────────────────────┘    └───────────────────────┘
      ┌────────────────┐
      │  QueryGovernor │
      │ + Observe(stat)│
      │ + Terminate(pid)│
      └────────────────┘
```

Relações: linha cheia com `▶` = dependência de uso; `△` = implementação de interface.

---

## 3. Contratos (interfaces públicas do serviço)

```
  interface IMigrationEngine
  ────────────────────────────────────────────────────────────────────────────
  + SubmitAsync(MigrationRequest, IdempotencyKey) : Migration      // T-01
  + ValidateAsync(Urn)                            : Migration      // T-02 | T-03
  + DryRunAsync(Urn)                              : Migration      // T-04 | T-05
  + ApplyAsync(Urn, IdempotencyKey)               : Migration      // T-06 → T-07|T-08
  + RollbackAsync(Urn, Reason)                    : Migration      // T-09
  + GetAsync(Urn) / ListAsync(Filter)             : Migration[]
  Invariantes: I1 (lock global), I2 (atomicidade), I5 (digest imutável), I6 (dry-run)

  interface ISchemaRegistry
  ────────────────────────────────────────────────────────────────────────────
  + GetAsync(schema, table)          : SchemaObject
  + UpsertAsync(SchemaObject)        : SchemaObject     // OCC por version
  + AuditAsync()                     : ComplianceReport // FR-001, FR-006, FR-019
  Invariante: multi_tenant = true ⇒ rls_enabled = true ∧ retention_ref ≠ null

  interface IRetentionService
  ────────────────────────────────────────────────────────────────────────────
  + PutPolicyAsync(RetentionPolicy)  : RetentionPolicy
  + RunCycleAsync()                  : RetentionOutcome
  + RequestErasureAsync(ErasureRequest, IdempotencyKey) : ErasureRequest
  Invariante: legal_hold = true ⇒ nenhuma remoção (AIOS-DB-0010)

  interface IDurabilityService
  ────────────────────────────────────────────────────────────────────────────
  + ListBackupsAsync()               : BackupRecord[]
  + VerifyAsync(Urn)                 : VerificationResult
  + RestorePointInTimeAsync(Target, IdempotencyKey) : RestoreOutcome
  + GetReplicationStatusAsync()      : ReplicationStatus
  Invariante: backup sem verified_at NÃO conta para RTO/RPO (FR-012)

  interface IVectorIndexService
  ────────────────────────────────────────────────────────────────────────────
  + PutAsync(VectorIndexSpec)        : VectorIndexSpec
  + ReindexAsync(schema, table, col) : ReindexOutcome
  Invariante: dimensions imutável após criação (AIOS-DB-0011)

  interface IPolicyClient    // PEP → PDP (022), default deny
  ────────────────────────────────────────────────────────────────────────────
  + DecideAsync(DecisionRequest)     : Decision   // Allow | Deny(code)
  Falha do PDP ⇒ Deny (fail-closed), erro AIOS-DB-0002
```

---

## 4. Estruturas de Dados (projeção das entidades)

```
  ┌──────────────────────────────┐        ┌───────────────────────────────┐
  │ SchemaObject                 │        │ Migration                     │
  │──────────────────────────────│        │───────────────────────────────│
  │ Urn            : Urn         │        │ Urn             : Urn         │
  │ SchemaName     : string      │  1   * │ Version         : string      │
  │ TableName      : string      │◀───────│ OwnerModule     : string      │
  │ OwnerModule    : string      │ afeta  │ State           : MigrationState
  │ MultiTenant    : bool        │        │ Phase           : {expand,     │
  │ RlsEnabled     : bool        │        │                    migrate,    │
  │ DataClass      : DataClass   │        │                    contract}   │
  │ PartitionStrat.: PartStrategy│        │ UpSqlDigest     : Sha256      │
  │ RetentionRef   : Guid?       │        │ DownSqlDigest   : Sha256?     │
  │ SchemaVersion  : string      │        │ Reversible      : bool        │
  │ RowEstimate    : long        │        │ BlockingEstimate: int (ms)    │
  │ BytesEstimate  : long        │        │ DryRunAt        : DateTime?   │
  │ Version        : long (OCC)  │        │ AppliedAt       : DateTime?   │
  └───────┬──────────────────────┘        │ AppliedBy       : Urn         │
          │ 1                              │ FailureCode     : string?     │
          │                                │ SupersededBy    : Urn?        │
          ├──────▶ RetentionPolicy 0..1     │ OccVersion     : long        │
          │        { Ttl, BasisColumn,      └───────────────────────────────┘
          │          Method, LegalHold,
          │          LegalBasis, BatchSize }
          │
          ├──────▶ PartitionPolicy 0..1
          │        { Strategy, Interval, HashModulus,
          │          PrecreateAhead, DetachBeforeDrop }
          │
          └──────▶ VectorIndexSpec 0..*
                   { ColumnName, Dimensions, IndexType,
                     DistanceOp, Params, TargetRecall }

  ┌──────────────────────────────┐        ┌───────────────────────────────┐
  │ ErasureRequest               │        │ BackupRecord                  │
  │──────────────────────────────│        │───────────────────────────────│
  │ Urn         : Urn            │        │ Urn        : Urn              │
  │ TenantId    : string  (RLS)  │        │ Kind       : {full,           │
  │ SubjectUrn  : Urn            │        │               incremental,    │
  │ Scope       : {subject,tenant}│       │               wal_segment}    │
  │ Status      : ErasureStatus  │        │ ObjectUri  : string           │
  │ TablesAffect: string[]       │        │ LsnStart   : PgLsn            │
  │ RowsErased  : long           │        │ LsnEnd     : PgLsn?           │
  │ ReceiptHash : Sha256?        │        │ Checksum   : Sha256           │
  └──────────────────────────────┘        │ VerifiedAt : DateTime?        │
                                          │ ExpiresAt  : DateTime         │
                                          └───────────────────────────────┘
```

Tipos `Urn`, `Sha256` e `PgLsn` são *value objects* imutáveis; `Urn` valida o formato
`urn:aios:<tenant>:<tipo>:<ULID>` conforme RFC-0001 §5.1 — o módulo **não redefine** o
formato, apenas o valida.

---

## 5. Invariantes de Estrutura

| # | Invariante | Onde é garantido |
|---|-----------|------------------|
| C-01 | `SchemaObject.MultiTenant = true` ⇒ `RlsEnabled = true` ∧ `RetentionRef ≠ null`. | `ISchemaRegistry.UpsertAsync` + auditoria contínua |
| C-02 | `Migration` em estado terminal é imutável, exceto `SupersededBy`. | `MigrationLedger` (guarda de escrita) |
| C-03 | `VectorIndexSpec.Dimensions` é imutável após a criação do índice. | `IVectorIndexService.PutAsync` → `AIOS-DB-0011` |
| C-04 | `RetentionPolicy.LegalHold = true` ⇒ nenhum caminho de código remove linhas dessa tabela. | `RetentionEnforcer` e `ErasureCoordinator` |
| C-05 | Toda mutação de entidade de plataforma incrementa `Version` (OCC); conflito → `AIOS-DB-0004`. | Camada de persistência |
| C-06 | `BackupRecord.VerifiedAt = null` ⇒ o registro não é elegível para plano de recuperação. | `IDurabilityService` |

---

## 6. Referências

- Componentes e diagrama-fonte: `./_DESIGN_BRIEF.md` §2
- Arquitetura: `./Architecture.md` · FSM: `./StateMachine.md`
- Modelo físico: `./Database.md` · API: `./API.md`
